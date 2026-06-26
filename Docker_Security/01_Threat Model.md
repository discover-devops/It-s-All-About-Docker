# Topic 1: Threat Model

This is the foundation of the entire session. In my opinion, this is the single most important topic in the course because every security control that follows exists to solve one of the problems introduced here. If students don't understand the threat model, the remaining chapters become a list of commands that they memorize instead of security concepts they understand.

The content itself is technically correct and I would not remove anything from it. 

However, I would change **how** it is introduced.

I would not begin by drawing the attack diagram immediately. Instead, I would first ask the class a simple question:

> "Suppose your company has deployed a banking application inside a Docker container. The application has a Remote Code Execution vulnerability and an attacker gains shell access to the container. Is the attack over?"

Most beginners will answer "Yes, the attacker has compromised the system."

That is where you introduce the central idea of Docker security. Explain that compromising a container does **not** automatically mean compromising the entire server. The attacker still has multiple barriers to cross. They must try to escape the container, interact with the Docker daemon, exploit the host kernel, or move laterally to other containers. Docker security is about making each of those steps as difficult as possible.

Now the attack diagram becomes meaningful instead of being just another picture.

When you explain the attack flow, don't rush through the boxes. Spend a minute on each layer.

The **Internet** represents an untrusted source from where attacks originate. The **Exposed Ports** are the application's front door. If an application listening on port 8080 has a vulnerability, that becomes the initial entry point. The **Container** is where the attacker first lands after exploiting the application. The **Container Runtime** represents the isolation boundary implemented using Linux namespaces and cgroups. The **Docker Daemon** is the management service that controls all containers on the host. Finally, the **Host Operating System** is the ultimate target because once an attacker controls the host, every container on that machine is effectively compromised.

Students often misunderstand the difference between the container runtime and the Docker daemon. This is a good opportunity to remind them that the runtime is responsible for actually creating and managing isolated processes, whereas the Docker daemon is the management service that receives commands from the Docker CLI and orchestrates container lifecycle. They work together but they are different components.

When you discuss the security goals—prevent initial compromise, contain the compromise, detect suspicious activity, and minimize damage—I would relate each one to a practical scenario instead of reading the list. For example, preventing compromise means reducing the attack surface by exposing only required ports. Containment means ensuring that even if one container is compromised, the attacker cannot escape to the host. Detection means monitoring logs and runtime events so that suspicious activity is noticed quickly. Minimizing damage means following the principle of least privilege so that a compromised process has very limited permissions.

The list of common attack vectors is complete and technically accurate. I would not explain every attack in detail here because each one becomes a chapter later in the course. Instead, introduce each attack in one or two sentences and tell the students that the rest of the session will focus on preventing these exact attacks. This creates continuity throughout the class.

I would add one sentence that is currently missing:

> "Every security control we learn today exists because of one or more of these attack vectors. Whenever we enable a security feature, ask yourself: which attack is this feature trying to stop?"

That sentence becomes a recurring theme throughout the session.

I don't think a full lab is required for this topic because this is primarily a conceptual foundation. However, if you want a very short demonstration to make it tangible, you can do the following:

1. Start an Ubuntu container:

   ```bash
   docker run -it --name demo ubuntu bash
   ```

2. Inside the container, run:

   ```bash
   whoami
   uname -a
   ps -ef
   ```

3. Explain that although the attacker now has a shell inside the container, they are still inside the container's isolated environment. The next challenge for the attacker is to escape that isolation and reach the host. That transition naturally leads into the Host-Level Security section.

This keeps the focus on understanding the attack journey rather than overwhelming students with security controls too early.
