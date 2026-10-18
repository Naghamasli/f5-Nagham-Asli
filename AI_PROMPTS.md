# Generative AI Disclosure

Using AI tools for this challenge is allowed. Be honest here — an accurate
"I used AI heavily" is worth more to us than an inaccurate "I did not".

**Did you use AI tools?** Yes

**Which ones?** ChatGPT (OpenAI)

---

## 1. Key prompts

The main prompts and questions I used during the challenge (paraphrased):

1. Help me analyze a Docker build failure, understand the full error output, and identify why Nginx installation fails during the image build.
2. Explain how Nginx upstream routing works, how to identify the destination from proxy logs, and why `127.0.0.1` does not reach another Docker container.
3. Help me investigate why integration tests fail during a fresh startup but pass when the backend and proxy containers are already running.
4. Explain the difference between Docker Compose `depends_on` and application readiness, and how health checks can prevent startup race conditions.
5. Help me investigate the GitHub Actions workflow, understand Linux executable permissions, and preserve the executable bit in Git.
6. Guide me through optimizing the proxy image, running Nginx as a non-root user, and implementing additional security improvements.
7. Help me review and organize my findings into a clear Root Cause Analysis (RCA) document.

---

## 2. Where the AI got it wrong

Initially, ChatGPT suggested running the integration tests using `bash ./scripts/run-tests.sh`. Although this worked locally, it bypassed the executable permission requirement and did not reproduce the command used by GitHub Actions. I identified the issue by comparing the workflow configuration with my local execution method and inspecting the script's permissions. I corrected the executable bit using `chmod +x`, ensured Git recorded mode `100755`, and verified that `./scripts/run-tests.sh` executed successfully without explicitly invoking Bash.

---

## 3. Anything in your submission you could not fully explain?

I used AI extensively as a debugging and learning assistant. I tested the suggested fixes locally, inspected Docker and Nginx logs, and verified the results rather than relying only on the AI's explanations. I reviewed the changes involving container networking, health checks, executable permissions, image optimization, and non-root execution.

I do not currently have any specific changes that I consider unexplained.
