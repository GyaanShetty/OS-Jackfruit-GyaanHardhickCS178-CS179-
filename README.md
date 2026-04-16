Multi-Container Runtime — OS-Jackfruit
Team Information
Name	SRN
Gyaan S Shetty	PES1UG24CS178
Hardhick M Gowda	PES1UG24CS179

Build, Load, and Run Instructions
Prerequisites

sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)

Build

cd boilerplate
make
Prepare Root Filesystems
mkdir -p rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta

# Copy workload binaries
cp cpu_hog memory_hog io_pulse ./rootfs-alpha/
cp cpu_hog memory_hog io_pulse ./rootfs-beta/
Load Kernel Module
sudo insmod monitor.ko
ls -l /dev/container_monitor   # verify device
sudo dmesg | tail -3           # verify load
Start Supervisor (Terminal 1)
sudo ./engine supervisor ./rootfs-base
Use CLI (Terminal 2)
# Start containers
sudo ./engine start alpha ./rootfs-alpha /cpu_hog --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  /cpu_hog --soft-mib 64 --hard-mib 96

# Run container (foreground)
sudo ./engine run alpha ./rootfs-alpha /memory_hog --soft-mib 20 --hard-mib 64

# List containers
sudo ./engine ps

# View logs
sudo ./engine logs alpha

# Stop container
sudo ./engine stop alpha
Cleanup
# Stop supervisor (Ctrl+C)
sudo rmmod monitor
sudo dmesg | tail -5
make clean
Demo Screenshots
1. Multi-container supervision
sudo ./engine start alpha ./rootfs-alpha /cpu_hog --nice 0
sudo ./engine start beta  ./rootfs-beta  /cpu_hog --nice 10
sudo ./engine ps
<img width="759" height="236" alt="Screenshot 2026-04-16 at 11 29 40 PM" src="https://github.com/user-attachments/assets/c06cacfe-7a13-445e-9c49-6f38dad7d3ec" />2. Metadata tracking
sudo ./engine ps
<img width="762" height="377" alt="Screenshot 2026-04-16 at 11 30 09 PM" src="https://github.com/user-attachments/assets/0ae13a7a-016c-452e-8c42-efc374ebf0aa" />

3. Logging system
sudo ./engine start alpha ./rootfs-alpha /cpu_hog
sleep 3
sudo ./engine logs alpha
<img width="754" height="90" alt="Screenshot 2026-04-16 at 11 29 50 PM" src="https://github.com/user-attachments/assets/252dded0-225e-440b-b45f-138cb2ad8f7d" />

4. CLI and IPC
sudo ./engine start alpha ./rootfs-alpha /cpu_hog --soft-mib 48 --hard-mib 80
sudo ./engine stop alpha
sudo ./engine ps
<img width="721" height="112" alt="Screenshot 2026-04-16 at 11 30 28 PM" src="https://github.com/user-attachments/assets/41133b40-0d44-4c6e-a539-454e54818a33" />

5. Soft-limit warning
sudo ./engine run alpha ./rootfs-alpha /memory_hog --soft-mib 20 --hard-mib 64
sudo dmesg | grep "SOFT LIMIT"
<img width="767" height="170" alt="Screenshot 2026-04-16 at 11 30 49 PM" src="https://github.com/user-attachments/assets/422f800f-8a12-4d9b-a412-73aa0134b41a" />

6. Hard-limit enforcement
sudo ./engine run alpha ./rootfs-alpha /memory_hog --soft-mib 10 --hard-mib 20
sudo dmesg | grep container_monitor | tail -5
sudo ./engine ps
<img width="757" height="342" alt="Screenshot 2026-04-16 at 11 31 10 PM" src="https://github.com/user-attachments/assets/0b3f391c-886e-47fb-a9eb-c08396a8a6cb" />

7. Scheduling experiment
sudo ./engine start alpha ./rootfs-alpha /cpu_hog --nice 0
sudo ./engine start beta  ./rootfs-beta  /cpu_hog --nice 10
top -d 1 -n 8
<img width="748" height="351" alt="Screenshot 2026-04-16 at 11 31 33 PM" src="https://github.com/user-attachments/assets/e718e312-cf33-4edb-88be-c2681bf32eda" />
<img width="756" height="234" alt="Screenshot 2026-04-16 at 11 31 27 PM" src="https://github.com/user-attachments/assets/14f880f1-66a3-493c-9329-1cd7b5358d8d" />

8. Clean teardown
ps aux | grep defunct
sudo rmmod monitor
sudo dmesg | tail -3
<img width="758" height="152" alt="Screenshot 2026-04-16 at 11 32 26 PM" src="https://github.com/user-attachments/assets/f61a6b63-ab84-4361-aa4c-72a44ef06b2f" />

Engineering Overview
Isolation
Uses Linux namespaces (PID, UTS, Mount)
Each container has its own root filesystem using chroot
Supervisor
Manages all containers
Handles start/stop
Cleans up processes (no zombies)
IPC & Logging
CLI ↔ Supervisor → UNIX socket
Logs → pipes → stored in files
Memory Management
Soft limit → warning
Hard limit → process killed
Checked inside kernel module
Scheduling
Controlled using nice values
Lower nice → more CPU
Higher nice → less CPU
Design Choices
chroot → simple filesystem isolation
Single supervisor → easy control
Pipes → efficient logging
UNIX socket → clean communication
Nice values → simple scheduling experiment
Scheduler Result
Container	NI	CPU
alpha	0	~100%
beta	10	~60%
Conclusion:
Lower nice value gets more CPU time.
