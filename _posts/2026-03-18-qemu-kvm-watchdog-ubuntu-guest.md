---
layout: post
title: "Setting up a QEMU/KVM Hardware Watchdog with an Ubuntu Guest"
date: 2026-03-18 16:00:00 -0400
categories: [linux, virtualization]
tags: [qemu, kvm, libvirt, ubuntu, watchdog, i6300esb]
---

Virtual machines lock up. It happens. Whether it's an out-of-memory condition, a deadlocked kernel, or just bad drivers, sometimes your guest OS becomes completely unresponsive while the hypervisor keeps merrily chugging along.

If you are running your VMs on QEMU/KVM via `libvirt`, you don't need a clunky host-side script to ping the guest and force a reboot. QEMU has a native hardware watchdog built right in. 

By passing a virtual watchdog device (`i6300esb`) to the VM, we can have a daemon inside the guest OS constantly reset a countdown timer. If the guest locks up and misses its deadline, QEMU natively catches the timeout and hard-resets the VM from the outside.

Here is how to set it up end-to-end with an Ubuntu guest.

## 1. Host-Side: Injecting the Virtual Hardware

First, we need to attach the virtual watchdog hardware to the VM. On your libvirt host (e.g., Debian), edit the VM configuration:

```bash
virsh edit my-vm-name
```

Find the `<devices>` section and add the watchdog device:

```xml
<devices>
  <!-- ... other devices ... -->
  <watchdog model='i6300esb' action='reset'/>
</devices>
```

The `action='reset'` tells QEMU to hard reset the VM if the timer expires. 

Save the XML, then completely shut down and restart the VM so QEMU can build the new virtual hardware:

```bash
virsh destroy my-vm-name
virsh start my-vm-name
```

## 2. Guest-Side: The Watchdog Daemon

Now, boot into your Ubuntu guest. The hardware exists, but the countdown timer won't arm until a process opens the `/dev/watchdog` device file.

Install the standard Linux watchdog daemon:

```bash
sudo apt update && sudo apt install watchdog -y
```

### The Race Condition Fix

There is a known gotcha with the `watchdog` package on Ubuntu: the daemon often starts *before* the kernel module (`i6300esb`) is loaded. If this happens, the daemon fails to find `/dev/watchdog` and aborts, leaving your VM completely unprotected.

To fix this race condition permanently, we need to tell the `watchdog` systemd unit to explicitly load the kernel module before it starts.

Edit `/etc/default/watchdog`:

```bash
sudo nano /etc/default/watchdog
```

Set the `watchdog_module` variable to the hardware driver:

```ini
# Start watchdog at boot time? 0 or 1
run_watchdog=1
# Start wd_keepalive after stopping watchdog? 0 or 1
run_wd_keepalive=1
# Load module before starting watchdog
watchdog_module="i6300esb"
```

Next, configure the daemon itself. Edit `/etc/watchdog.conf`:

```bash
sudo nano /etc/watchdog.conf
```

Uncomment or add these lines to configure the device and the timeout:

```ini
watchdog-device = /dev/watchdog
watchdog-timeout = 15
realtime = yes
priority = 1
```

Finally, restart the service so it picks up the new configuration and forces the module load:

```bash
sudo systemctl restart watchdog
sudo systemctl status watchdog
```

You should see an active status and a log line confirming the hardware is armed:

```text
watchdog[1234]: hardware watchdog identity: i6300ESB timer
```

## 3. The Live Fire Test

The absolute only way to prove the entire chain works—from the systemd daemon, to the kernel module, to the virtual PCI device, all the way up to the QEMU host process—is to crash the guest. 

**Warning: This will instantly lock up your VM.**

Trigger a kernel panic:

```bash
sudo bash -c 'echo c > /proc/sysrq-trigger'
```

Your SSH session will freeze instantly. The daemon is now dead and can no longer write to `/dev/watchdog`.

Wait about 15-20 seconds. QEMU will realize the timer expired, execute the `reset` action, and your VM will automatically begin rebooting.

Your VM is now protected against hard lockups.
