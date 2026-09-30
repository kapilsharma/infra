# Infra instructions.

There are different projects at the parent folder:

- GBPN
- GMTC
- LIQ
- oppe
- infra (Current folder)

Above four are four different products. They are still in development phase of a startup, and there is no income. All funding are personal fundings so need to keep costs minimal initialls.

Initially, they were vibe coded by claude design/claude code with main tech stack like react, hono, postgres and also have their own terraform.

However, maintaining two environments (test and live) for 4 projects seems could be little too costly initialls. I had that quick architecture chat with gemini and saved in [./gemini.md]()

Problem is, when things scale, we need to go with serverless/AWS lambda, but it comes with its own cost for test servers, live servers before first doller is earned. So, that discussion was about minimizing initial cost, specially on test server.

We might delay release of Oppe but other three products (GBPN, GMTC, LIQ) need to be released on test server for better testing. This project (infra) is supposed to be its own git repo and deliver infra for all above three products.

Go through above project in details, their requirements (by reading their README.md) and we need to decide best infra for now (without earning) but with later possibility for scaling.

We will make a docs folder and record all decision there.