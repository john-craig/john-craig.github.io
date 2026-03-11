---
layout: post
title: "Restricted Remote Access for AI Agents with lshell"
---
# Restricted Remote Access for AI Agents with `lshell`
AI-powered coding agents like Claude, Codex, and Bob are skyrocketing in popularity in recent months. Part of what is driving their mass adoption is the ability to extend their functionality through MCP servers, allowing for the creation of truly agentic workflows through different tools and interfaces.

What has been overlooked during this frenzy are guardrails and security controls. Although not as exciting, these are needed to ensure that these agents cannot utilize these tools to damage their environment, either as an unintended consequence of their own attempts at solving problems or due do malicious hijacking.

As with all things, MCP tools must maintain a balance between security and usability. A tool in which this balance is particularly nuanced is remote connections over secure shell. On the one hand, an agent with the ability to connect to remote systems is incredibly powerful; however, that power comes with a substantial risk. Allowing remote access increases the footprint of devices that an agent can potentially modify and thus, damage.

### `lshell`: The Limited Shell
One useful tool for restricting an AI agent's capabilities on a remote system is a Python package called `lshell`. `lshell` is a user shell, like `bash` or `zsh`, but what makes it unique is that it can be configured to allow only a specific list of commands to be executed.

This means that by creating a dedicated user for our AI agent and changing their shell to `lshell`, we can control exactly which commands the AI agent is capable of executing on the remote server. If we wanted our agent to be able to retrieve system diagnostics, for example, we might permit a set of commands like `netstat`, for seeing which ports are open, or `top`, to see which processes are using the most resources.

```
TODO: example
```

We can also combine these per-command restrictions at the shell level with the existing per-command privilege escalation restrictions provided by `sudo`. For example, if we wanted our agent to retrieve the status of `systemd` units, as well as logs from the system journal, we might create a `/etc/sudoers` entry like so:
```sh
TODO: example
```

These kinds of configurations allow our AI agents to safely and securely retrieve information from a remote system without the risk of making unintended modifications while doing so. 

I hope you found this article useful, and it allows you to make your agentic environment a little more secure. :)