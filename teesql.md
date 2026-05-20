```
 ▐▛███▜▌   Claude Code v2.1.145
▝▜█████▛▘  Opus 4.7 (1M context) with high effort · Claude Max
  ▘▘ ▝▝    ~/Code/research
```

❯ can you review https://teesql.com/ and tell me how i would build a web
  application on this? i'm thinking just an anonymous forum where every user/login
  is assigned a persona (eg "pink crab" or "orange flamingo") and retains that
  persona. can we encode that business logic in the db?

⏺ Fetch(https://teesql.com/)
  ⎿  Received 38.2KB (200 OK)
  ⎿  Allowed by auto mode classifier

⏺ What TeeSQL is

  It's just PostgreSQL 17 in a hardware enclave (Intel TDX). No special API, no
  Supabase-style auto-REST, no built-in user auth. The differentiator is that
  operators (including TeeSQL's own team) can't see plaintext data — connections use
   mutual RA-TLS where both sides cryptographically attest to running in a TEE.

  So "how do you build on it" = "how do you build any Postgres app," plus one
  wrinkle around the client side.

  Architecture for your forum

  Browser ──HTTPS──> Backend (in a TEE) ──RA-TLS──> TeeSQL
                     sessions, signup,            users, posts,
                     rate limits                  personas

  The wrinkle: to preserve TeeSQL's "no one can see your data" guarantee end-to-end,
   the backend itself must run in a TEE (e.g., dstack on Phala). If it runs on a
  normal VPS, the DB operator still can't see data, but you (or anyone who pops your
   backend) can. For an anonymous forum that's a real consideration — the persona
  pseudonymity is only as strong as the place that holds the user_id → cookie
  mapping.

  You can run a normal Node/Go/Python backend, you just give up the strongest part
  of the pitch.

  Encoding the persona logic in the DB

  Yes, this is a great fit for Postgres — no app code required to assign personas.
  Two reasonable shapes:

  Option A: pre-seeded pool (guarantees uniqueness, finite supply)

  CREATE TABLE personas (
    id serial PRIMARY KEY,
    adjective text NOT NULL,
    animal text NOT NULL,
    claimed_by uuid REFERENCES users(id),
    UNIQUE (adjective, animal)
  );

  CREATE TABLE users (                                                                  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    login_token_hash bytea NOT NULL UNIQUE,                                             persona_id int NOT NULL REFERENCES personas(id) UNIQUE,
    created_at timestamptz DEFAULT now()
  );
  CREATE FUNCTION claim_persona() RETURNS int AS $$
    UPDATE personas SET claimed_by = NULL  -- placeholder, see trigger                  WHERE id = (
      SELECT id FROM personas                                                             WHERE claimed_by IS NULL
      ORDER BY random() LIMIT 1
      FOR UPDATE SKIP LOCKED                                                            )
    RETURNING id;
                                                                     $$ LANGUAGE sql;

  Then a BEFORE INSERT trigger on users calls claim_persona() and sets
  NEW.persona_id, and an AFTER INSERT sets claimed_by = NEW.id. FOR UPDATE SKIP
  LOCKED handles concurrent signups cleanly.
  Option B: generate on the fly (infinite supply, tiny collision risk)
  Two tables of adjectives + animals, pick random rows in a trigger, retry on
  unique-constraint conflict. Simpler, but you can collide once the namespace fills.

  Either way, the persona is set in the trigger so the backend literally cannot
  assign it — and once written it never changes because there's no UPDATE path
  exposed.

  Posting

  CREATE TABLE posts (
    id bigserial PRIMARY KEY,
    user_id uuid NOT NULL REFERENCES users(id),
    body text NOT NULL,
    created_at timestamptz DEFAULT now()
  );

  CREATE VIEW posts_public AS
  SELECT p.id, p.body, p.created_at,
         pe.adjective || ' ' || pe.animal AS author
  FROM posts p
  JOIN users u ON u.id = p.user_id
  JOIN personas pe ON pe.id = u.persona_id;

  Read-only clients only ever touch posts_public. You can lock this down hard with
  RLS + a forum_reader role that has SELECT on the view and nothing else, so even
  with a compromised backend nobody can pivot to users.login_token_hash.

  What you'd actually need to build

  1. A backend (any language) with a Postgres driver that supports client certs
  (RA-TLS is just mTLS with extra attestation steps).
  2. A login flow — for true anonymity, a one-time signup that issues a long random
  cookie token, hashed in the DB. No email needed.
  3. The schema above.
  4. A frontend.

  Honest caveats

  - TeeSQL is in early access, no pricing. Don't bet a launch on it yet.
  - The strong anonymity story requires your whole stack in a TEE, which is more
  setup than typical web hosting.
  - If you don't need confidential compute, a normal managed Postgres (Neon,
  Supabase, RDS) gets you the same forum at a fraction of the operational
  complexity. TeeSQL only earns its keep if "even I can't read user data" is a
  feature you're selling to users.