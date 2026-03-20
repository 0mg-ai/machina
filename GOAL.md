# Goal

I want to have an autonomous AI agent solution, to which I can speak via slack messenger, give him long lasting tasks, observe the progress, intercept, see results.

If it make sense, take inspiration from OpenClaw. https://docs.openclaw.ai/start/getting-started

Now I guess I need some kind of Gateway, and isolation for AI agent to run.


## HW

I want to create a design for AI agent solution, that I can have deployed on my Asus NUC - ASUS NUC 15 Pro - 32 GB - 1TB HDD. 

## Orchestration

Ideally I want to use microk8s/k8s. I see an advantage if I will be able to integrate it with it. Am I right?

Should I use Argo Workflows? 

## Isolated environment

Idea was to have a openshell (https://docs.nvidia.com/openshell/index.html) as the isolation layer.

I see the potential in openshell, being able to limit some of the networking ... etc. for the agent, but I don't like it is not integrable into microk8s.

Should I rather use "Agent Sandbox" - https://agent-sandbox.sigs.k8s.io/?


## AI Agents

As a AI agent to use either Claude Code (or Claude Agent SDK) or Langchain DeepAgent (https://docs.langchain.com/oss/python/deepagents/overview).

Each agent will be configurate via multiple markdown files. In case of Claude code, a plugins, skills, system message ...etc.

I also want to be able to define a team of agents. I think it would be good idea to be able to use some workflow system, where I can define any given complicated workflow I want, ex. temporal.io, prefect or something.


Should I use as runtime the gVisor - https://gvisor.dev/ ?


## AI Gateway

I don't know what to use as a AI agent gateway. I am not even sure I need one.


## Slack integration

The integration should be do with slack, so I can give "tasks" to the AI agents. Each slack channel will be able to "mention" any type of agent that I will have "defined" in the NUC. When the agent will get mentioned, he will start doing the task and output the progress into the slack message (either as comment), or as an answer that message.

# Questions

Propose an architecture and libraries that I should use with respect of my requirements.