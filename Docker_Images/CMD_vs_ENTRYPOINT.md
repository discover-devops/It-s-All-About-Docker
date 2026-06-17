# CMD vs ENTRYPOINT
# Context, Concept, and Step-by-Step Lab

---

## Context — Why This Confuses Everyone

CMD and ENTRYPOINT both answer the same question:

> "What runs when a container starts?"

Because they answer the same question, engineers assume they are interchangeable. They are not.

They have different jobs. They behave differently when you pass arguments to `docker run`. Understanding the difference changes how you design containers.

By the end of this document you will be able to answer:

- What does CMD do?
- What does ENTRYPOINT do?
- What happens when you combine them?
- When should you use which?

And you will have run every scenario yourself in the lab.

---

# Section 1: The Core Difference in One Sentence

**CMD** — defines the default command. Can be completely replaced at runtime.

**ENTRYPOINT** — defines the fixed executable. Arguments at runtime are appended to it, not replace it.

---

# Section 2: CMD in Detail

## Concept

CMD sets the default command that runs when a container starts.

The word **default** is the key. A default is something that works if you do not specify otherwise. But if you do specify otherwise, the default is ignored completely.

```dockerfile
FROM ubuntu
CMD ["echo", "Hello from CMD"]
```

Run with no arguments — default runs:

```bash
docker run myimage
# Output: Hello from CMD
```

Run with your own command — CMD is completely replaced:

```bash
docker run myimage echo "I replaced CMD"
# Output: I replaced CMD
```

Run with something entirely different — CMD is gone:

```bash
docker run myimage ls /etc
# Output: contents of /etc
# echo "Hello from CMD" never ran
```

CMD is a suggestion. It says "run this if nothing else is specified." The moment you pass any command to `docker run`, CMD is discarded entirely.

---

# Section 3: ENTRYPOINT in Detail

## Concept

ENTRYPOINT sets the fixed executable that always runs. It cannot be replaced by passing arguments to `docker run`. Arguments you pass to `docker run` are **appended** to ENTRYPOINT, not substituted for it.

```dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
```

Run with no arguments:

```bash
docker run myimage
# Output: (blank line — echo ran with no arguments)
```

Run with arguments — arguments are APPENDED to echo:

```bash
docker run myimage "Hello from ENTRYPOINT"
# Output: Hello from ENTRYPOINT
# What actually ran: echo "Hello from ENTRYPOINT"
```

Run with different arguments:

```bash
docker run myimage "Build Automate Architect"
# Output: Build Automate Architect
# What actually ran: echo "Build Automate Architect"
```

Notice: you cannot escape `echo`. The container always runs `echo`. You can only control what gets passed to it.

ENTRYPOINT makes the container behave like an executable. The container IS the command. You just control the arguments.

---

# Section 4: CMD and ENTRYPOINT Together

## Concept

This is where the real power is.

When you use both together:

- **ENTRYPOINT** = the fixed executable (never changes)
- **CMD** = the default arguments (can be replaced)

```dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
CMD ["Hello, this is the default message"]
```

Run with no arguments — CMD provides default arguments to ENTRYPOINT:

```bash
docker run myimage
# What ran: echo "Hello, this is the default message"
# Output:   Hello, this is the default message
```

Run with your own arguments — CMD is replaced, ENTRYPOINT stays:

```bash
docker run myimage "I replaced CMD but ENTRYPOINT stays"
# What ran: echo "I replaced CMD but ENTRYPOINT stays"
# Output:   I replaced CMD but ENTRYPOINT stays
```

Think of it this way:

```
ENTRYPOINT = the verb     (always runs)
CMD        = the noun     (default object, replaceable)

Together:  verb + noun
docker run: verb + your-noun  (you replaced the noun, verb stays)
```

---

# Section 5: The Behaviour Table

This table shows exactly what happens in every combination:

| Dockerfile has | docker run command | What actually runs |
|---------------|-------------------|-------------------|
| CMD ["echo","hi"] | (nothing) | echo hi |
| CMD ["echo","hi"] | ls /etc | ls /etc |
| ENTRYPOINT ["echo"] | (nothing) | echo |
| ENTRYPOINT ["echo"] | hi | echo hi |
| ENTRYPOINT ["echo"] CMD ["hi"] | (nothing) | echo hi |
| ENTRYPOINT ["echo"] CMD ["hi"] | bye | echo bye |
| ENTRYPOINT ["echo"] CMD ["hi"] | ls /etc | echo ls /etc |

The last row is important. With ENTRYPOINT set, even `ls /etc` becomes an argument to `echo`. The output would literally be `ls /etc` as a string — not a directory listing. This is why ENTRYPOINT is powerful but must be used carefully.

---

# Section 6: When to Use Which

| Use | When |
|-----|------|
| CMD only | Container has no fixed purpose. You want full flexibility. |
| ENTRYPOINT only | Container has one fixed job. Arguments may vary. |
| ENTRYPOINT + CMD | Container has one fixed job with a sensible default argument. |

**Real world examples:**

```
nginx container     → ENTRYPOINT ["nginx"] CMD ["-g","daemon off;"]
                      nginx always runs, config flags can be overridden

python script       → ENTRYPOINT ["python3"] CMD ["app.py"]
                      python3 always runs, script can be swapped

shell utility       → CMD ["bash"]
                      default is bash but can be replaced with anything

database backup     → ENTRYPOINT ["pg_dump"] CMD ["mydb"]
                      pg_dump always runs, database name is replaceable
```

---

# Section 7: The Override Escape Hatch

ENTRYPOINT seems unbreakable. But there is one way to override it — the `--entrypoint` flag:

```bash
docker run --entrypoint bash myimage
```

This replaces ENTRYPOINT entirely. Used for debugging containers that have ENTRYPOINT set to an application — you can still get a shell inside them.

---

# Section 8: Shell Form vs Exec Form

Both CMD and ENTRYPOINT have two syntaxes. This matters more than most people realise.

**Exec form — recommended:**

```dockerfile
CMD ["python3", "app.py"]
ENTRYPOINT ["python3"]
```

Written as a JSON array. Docker runs the command directly. The process becomes PID 1 inside the container. Signals like SIGTERM are delivered directly to the process.

**Shell form — avoid in production:**

```dockerfile
CMD python3 app.py
ENTRYPOINT python3
```

Written as a plain string. Docker runs this as:

```bash
/bin/sh -c "python3 app.py"
```

The shell becomes PID 1. Your process runs as a child of the shell. When Docker sends SIGTERM to stop the container, it goes to the shell (PID 1), not to your process. The shell may not forward it. Your application does not get a chance to shut down gracefully. Docker waits 10 seconds then sends SIGKILL. Data loss is possible.

**Always use exec form for CMD and ENTRYPOINT.**

---

---

# LAB: CMD vs ENTRYPOINT — See Every Behaviour Yourself

---

## Lab Setup

```bash
mkdir -p /home/cmd-entrypoint-lab
cd /home/cmd-entrypoint-lab
```

---

## Lab 1: CMD Alone — Default That Can Be Replaced

Create the Dockerfile:

```bash
cat > Dockerfile.cmd << 'EOF'
FROM ubuntu:22.04
CMD ["echo", "I am the CMD default"]
EOF
```

Build:

```bash
docker build -f Dockerfile.cmd -t test:cmd .
```

**Test 1 — Run with no arguments. CMD runs.**

```bash
docker run --rm test:cmd
```

Expected output:

```
I am the CMD default
```

**Test 2 — Run with your own command. CMD is replaced completely.**

```bash
docker run --rm test:cmd echo "I replaced CMD"
```

Expected output:

```
I replaced CMD
```

**Test 3 — Run an entirely different command. CMD is gone.**

```bash
docker run --rm test:cmd ls /etc
```

Expected output: contents of /etc directory. CMD never ran.

**Test 4 — Run with bash. CMD is gone.**

```bash
docker run --rm -it test:cmd bash
```

You get a shell. CMD never ran. Type `exit` to leave.

**Observation:** CMD is completely replaceable. The moment you pass anything to `docker run`, CMD is discarded.

---

## Lab 2: ENTRYPOINT Alone — Fixed Executable

```bash
cat > Dockerfile.entrypoint << 'EOF'
FROM ubuntu:22.04
ENTRYPOINT ["echo"]
EOF
```

Build:

```bash
docker build -f Dockerfile.entrypoint -t test:entrypoint .
```

**Test 1 — Run with no arguments.**

```bash
docker run --rm test:entrypoint
```

Expected output: blank line. `echo` ran with no arguments.

**Test 2 — Run with arguments. Arguments append to ENTRYPOINT.**

```bash
docker run --rm test:entrypoint "Hello from ENTRYPOINT"
```

Expected output:

```
Hello from ENTRYPOINT
```

What actually ran: `echo "Hello from ENTRYPOINT"`

**Test 3 — Try to run a different command. Watch what happens.**

```bash
docker run --rm test:entrypoint ls /etc
```

Expected output:

```
ls /etc
```

This is NOT a directory listing. `ls /etc` became a string argument to `echo`. The output is literally the text `ls /etc`. ENTRYPOINT cannot be replaced by passing arguments.

**Test 4 — The only way to override ENTRYPOINT.**

```bash
docker run --rm --entrypoint ls test:entrypoint /etc
```

Expected output: actual directory listing of /etc. The `--entrypoint` flag is the only escape.

**Observation:** ENTRYPOINT is fixed. Arguments you pass are appended to it. You cannot replace it through normal `docker run` arguments.

---

## Lab 3: ENTRYPOINT + CMD Together — Fixed Executable with Default Arguments

```bash
cat > Dockerfile.both << 'EOF'
FROM ubuntu:22.04
ENTRYPOINT ["echo"]
CMD ["I am the default argument to echo"]
EOF
```

Build:

```bash
docker build -f Dockerfile.both -t test:both .
```

**Test 1 — Run with no arguments. ENTRYPOINT + CMD runs together.**

```bash
docker run --rm test:both
```

Expected output:

```
I am the default argument to echo
```

What actually ran: `echo "I am the default argument to echo"`

**Test 2 — Run with your own argument. CMD is replaced, ENTRYPOINT stays.**

```bash
docker run --rm test:both "I replaced CMD but echo stays"
```

Expected output:

```
I replaced CMD but echo stays
```

What actually ran: `echo "I replaced CMD but echo stays"`

**Test 3 — Run with multiple arguments.**

```bash
docker run --rm test:both one two three
```

Expected output:

```
one two three
```

What actually ran: `echo one two three`

**Observation:** ENTRYPOINT always runs. CMD provides the default argument. Your runtime arguments replace CMD, not ENTRYPOINT.

---

## Lab 4: Real Application Pattern — Python Script Container

This is the most practical pattern. Container runs a Python script. Default script can be swapped.

Create a simple Python script:

```bash
cat > app.py << 'EOF'
import sys
import datetime

script_name = sys.argv[0] if len(sys.argv) > 0 else "app.py"
print(f"Running: {script_name}")
print(f"Time: {datetime.datetime.now().isoformat()}")
print(f"Args: {sys.argv[1:]}")
EOF
```

Create a second script:

```bash
cat > report.py << 'EOF'
import sys
import os

print("Running report script")
print(f"Environment: {os.environ.get('APP_ENV', 'unknown')}")
print(f"Args passed: {sys.argv[1:]}")
EOF
```

Create the Dockerfile:

```bash
cat > Dockerfile.python << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY app.py .
COPY report.py .

# python3 always runs
# app.py is the default script
# Any argument to docker run replaces app.py
ENTRYPOINT ["python3"]
CMD ["app.py"]
EOF
```

Build:

```bash
docker build -f Dockerfile.python -t test:python .
```

**Test 1 — Run with no arguments. python3 app.py runs.**

```bash
docker run --rm test:python
```

Expected output:

```
Running: app.py
Time: 2024-...
Args: []
```

**Test 2 — Run report.py instead. CMD is replaced, python3 stays.**

```bash
docker run --rm test:python report.py
```

Expected output:

```
Running report script
Environment: unknown
Args passed: []
```

**Test 3 — Pass arguments to the script.**

```bash
docker run --rm -e APP_ENV=production test:python report.py --verbose --output json
```

Expected output:

```
Running report script
Environment: production
Args passed: ['--verbose', '--output', 'json']
```

**Test 4 — Get a Python shell instead.**

```bash
docker run --rm -it test:python
```

You are now inside the Python interactive interpreter. `python3` is ENTRYPOINT so it ran — but with no arguments, it opens the REPL.

Type `exit()` to leave.

**Test 5 — Override ENTRYPOINT to get bash for debugging.**

```bash
docker run --rm -it --entrypoint bash test:python
```

You are now in bash inside the container. Useful for debugging.

---

## Lab 5: Shell Form vs Exec Form — See the Difference

This lab proves why exec form matters.

**Shell form Dockerfile:**

```bash
cat > Dockerfile.shell << 'EOF'
FROM ubuntu:22.04
CMD echo "I am shell form"
EOF
```

**Exec form Dockerfile:**

```bash
cat > Dockerfile.exec << 'EOF'
FROM ubuntu:22.04
CMD ["echo", "I am exec form"]
EOF
```

Build both:

```bash
docker build -f Dockerfile.shell -t test:shell .
docker build -f Dockerfile.exec -t test:exec .
```

Run both — output looks the same:

```bash
docker run --rm test:shell
docker run --rm test:exec
```

Now check PID 1 in each container:

```bash
echo "--- Shell form PID 1 ---"
docker run --rm test:shell ps -ef

echo ""
echo "--- Exec form PID 1 ---"
docker run --rm test:exec ps -ef
```

In shell form you will see:

```
PID 1: /bin/sh -c echo "I am shell form"
PID 7: echo I am shell form
```

Shell is PID 1. echo is a child process.

In exec form you will see:

```
PID 1: echo I am exec form
```

echo IS PID 1. No shell involved.

**Why this matters:**

```bash
cat > Dockerfile.signal-demo << 'EOF'
FROM ubuntu:22.04

# Shell form — SIGTERM goes to shell, not to sleep
# CMD sleep 300

# Exec form — SIGTERM goes directly to sleep
CMD ["sleep", "300"]
EOF

docker build -f Dockerfile.signal-demo -t test:signal .
```

Run container and time how long docker stop takes:

```bash
# First with exec form (CMD ["sleep", "300"])
docker run -d --name signal-test test:signal
time docker stop signal-test
docker rm signal-test
```

Exec form: `docker stop` completes in under 1 second. sleep received SIGTERM directly and exited.

Now change to shell form, rebuild, and test:

```bash
sed -i 's/CMD \["sleep", "300"\]/CMD sleep 300/' Dockerfile.signal-demo
docker build -f Dockerfile.signal-demo -t test:signal .
docker run -d --name signal-test test:signal
time docker stop signal-test
docker rm signal-test
```

Shell form: `docker stop` takes 10 seconds. The shell received SIGTERM but did not forward it to sleep. Docker waited 10 seconds then sent SIGKILL.

**10 seconds vs under 1 second. This is the exec form vs shell form difference made visible.**

---

## Lab 6: The Practical Summary Lab

Build one final container that shows all patterns working together:

```bash
cat > Dockerfile.summary << 'EOF'
FROM python:3.11-slim

WORKDIR /app

RUN pip install --no-cache-dir flask

COPY app.py .

# ENTRYPOINT: python3 always runs
# CMD: app.py is the default
# To run a different script: docker run myimage other.py
# To debug: docker run --entrypoint bash myimage
ENTRYPOINT ["python3"]
CMD ["app.py"]
EOF
```

Build:

```bash
docker build -f Dockerfile.summary -t test:summary .
```

Run the complete test sequence:

```bash
echo "=== Test 1: Default behaviour ==="
docker run --rm test:summary

echo ""
echo "=== Test 2: Different script (CMD replaced) ==="
docker run --rm test:summary report.py

echo ""
echo "=== Test 3: Python one-liner (CMD replaced) ==="
docker run --rm test:summary -c "print('inline python')"

echo ""
echo "=== Test 4: Python version (CMD replaced) ==="
docker run --rm test:summary --version

echo ""
echo "=== Test 5: Debug with bash (ENTRYPOINT overridden) ==="
docker run --rm --entrypoint bash test:summary -c "echo I am in bash && python3 --version"
```

---

## Lab Cleanup

```bash
docker rmi test:cmd test:entrypoint test:both test:python test:shell test:exec test:signal test:summary 2>/dev/null
docker system prune -f
```

---

# Full Reference Summary

## Behaviour Matrix

| | No args | Runtime args | --entrypoint flag |
|-|---------|-------------|------------------|
| CMD only | CMD runs | CMD replaced | ENTRYPOINT runs, CMD replaced |
| ENTRYPOINT only | ENTRYPOINT runs | Args appended | New ENTRYPOINT runs |
| Both | ENTRYPOINT + CMD | ENTRYPOINT + your args | New ENTRYPOINT + your args |

## Decision Guide

```
Q: Should the container always run the same program?
   YES → use ENTRYPOINT
   NO  → use CMD

Q: Should the container have a sensible default but allow swapping?
   YES → use ENTRYPOINT + CMD

Q: Is the container a general purpose base?
   YES → use CMD only

Q: Always exec form or shell form?
   ALWAYS exec form → ["command", "arg1", "arg2"]
```

## The Three Rules

```
Rule 1: CMD is a default — replaced entirely by docker run arguments
Rule 2: ENTRYPOINT is fixed — docker run arguments are appended to it
Rule 3: Together: ENTRYPOINT=verb, CMD=default noun
        docker run args replace the noun, verb always stays
```

---

*End of Document — CMD vs ENTRYPOINT*
