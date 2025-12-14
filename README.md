# 3ATIVE's findings for optimizing UnRaid 10GbE Network Performance

## From 30 MB/s to 800+ MB/s 🚀

---

## 📋 Table of Contents

- [The Problem](#-the-problem)
- [The Solution](#-the-solution)
- [Part 1: Enable Jumbo Frames](#part-1-enable-jumbo-frames)
  - [On UnRaid](#on-UnRaid)
  - [On Windows 11](#on-windows-11)
  - [Verify Jumbo Frames](#verify-jumbo-frames-are-working)
- [Part 2: Keep SMB Config Simple](#part-2-keep-smb-config-simple-critical)
- [Part 3: SMB Security Settings](#part-3-smb-security-settings-optional)
- [Expected Results](#-expected-results)
- [Troubleshooting](#-troubleshooting)
- [Why This Works](#-why-this-works)
- [Configuration Checklist](#-final-configuration-checklist)
- [Hardware Tested](#-hardware-tested)

---

## 🔴 The Problem

| Issue | Speed | Expected |
|-------|-------|----------|
| **Write speeds** | 30-70 MB/s | 500-800+ MB/s |
| **Read speeds** | 80 MB/s | 200-250+ MB/s |
| **Goal** | Maximize 10 Gigabit Ethernet performance | ✓ |

---

## ✅ The Solution

**Jumbo Frames + Default SMB Configuration**

| Metric | Result |
|--------|--------|
| **Write speeds** | 800 MB/s |
| **Read speeds** | 222 MB/s |
| **Improvement** | **25x faster!** 🎉 |

---

## Part 1: Enable Jumbo Frames

Jumbo frames allow larger network packets (9000+ bytes vs 1500 bytes), dramatically reducing overhead on high-speed networks.

### On UnRaid

1. Navigate to **Settings → Network Settings**
2. Find your network interface (typically `eth0`)
3. Set **Desired MTU: 9000** (or 9014)
4. Click **Apply**
5. **Reboot UnRaid**

#### Verify it worked:

```bash
ip addr show eth0
ip addr show br0
```

Both should show `mtu 9000` (or `mtu 9014`)

---

### On Windows 11

1. Press **Windows Key + X** → **Device Manager**
2. Expand **Network adapters**
3. Right-click your **10GbE adapter** → **Properties**
4. Click **Advanced** tab
5. Find **Jumbo Packet** or **Jumbo Frame**
6. Set to **9014 Bytes** (or 9000 if 9014 unavailable)
7. Click **OK**

#### Enable network offloading features (same Advanced tab):

- ✅ **Large Send Offload V2 (IPv4)** = Enabled
- ✅ **Large Send Offload V2 (IPv6)** = Enabled
- ✅ **Receive Side Scaling** = Enabled
- ✅ **IPv4 Checksum Offload** = RX & TX Enabled
- ✅ **TCP Checksum Offload (IPv4)** = RX & TX Enabled
- ✅ **TCP Checksum Offload (IPv6)** = RX & TX Enabled
- ✅ **UDP Checksum Offload (IPv4)** = RX & TX Enabled
- ✅ **UDP Checksum Offload (IPv6)** = RX & TX Enabled

#### Disable power saving:

- Go to **Power Management** tab
- **UNCHECK** "Allow the computer to turn off this device to save power"
- Click **OK**

**Reboot Windows**

---

### Verify Jumbo Frames Are Working

From Windows Command Prompt:

```cmd
ping [your-UnRaid-ip] -f -l 8972
```

**✅ Success:**
```
Reply from [IP]: bytes=8972 time<1ms TTL=64
```

**❌ Failure:**
```
Packet needs to be fragmented but DF set
```

---

## Part 2: Keep SMB Config Simple (CRITICAL!)

> **🔑 The Key Discovery:** Manual SMB "optimizations" actually **hurt** performance by disabling Linux's intelligent TCP auto-tuning.

### On UnRaid:

1. **Settings → SMB**
2. Check **SMB Extras** section
3. **Keep it empty** or minimal - do NOT add socket buffer settings

#### ❌ What NOT to add:

```bash
# DON'T USE THESE - They disable kernel auto-tuning!
socket options = IPTOS_LOWDELAY SO_RCVBUF=524288 SO_SNDBUF=524288
```

> **Important:** If you have custom SMB settings, remove them and test with defaults first.

4. Click **Apply**
5. Restart SMB:

```bash
/etc/rc.d/rc.samba restart
```

---

## Part 3: SMB Security Settings (Optional)

For best security + performance balance:

### On Windows (PowerShell as Admin):

```powershell
Set-SmbClientConfiguration -EnableSecuritySignature $true -Force
Set-SmbClientConfiguration -RequireSecuritySignature $false -Force
```

This enables SMB signing for security but doesn't require it, allowing flexibility.

---

## 📊 Expected Results

| Storage Type | Write to Cache | Read from Array |
|--------------|----------------|-----------------|
| **SATA SSD Cache** | 500-550 MB/s | 200-250 MB/s |
| **NVMe Cache** | 800-1,100 MB/s | 200-250 MB/s |

> *Note: Read speeds are limited by spinning disk speed*

---

## 🔧 Troubleshooting

### Still getting slow speeds?

#### Check your network switch:

- ✅ Must support jumbo frames (MTU 9000+)
- ✅ Many unmanaged switches support this automatically
- ✅ Verify your switch specs

#### Check cache drive settings:

- **Shares → [Your Share] → Primary storage:** Set to **Prefer** (uses cache first)
- Verify cache pool is assigned

#### Check actual link speed:

```bash
ethtool eth0 | grep Speed
```

Should show: `Speed: 10000Mb/s`

---

## 💡 Why This Works

### Jumbo Frames:

- **Standard packets:** 1500 bytes = massive overhead at 10 Gbps
- **Jumbo frames:** 9000 bytes = minimal overhead
- **Result:** 25x+ improvement in write speeds

### Default SMB Config:

- Modern Linux kernels have **intelligent TCP buffer auto-tuning**
- Manual buffer sizes **disable** this auto-tuning
- **Default = Best** for modern systems

### SMB Signing:

- Contrary to expectations, signing doesn't significantly impact performance on modern hardware
- Keep it enabled for security on home networks

---

## ✅ Final Configuration Checklist

- [ ] Jumbo frames enabled on UnRaid (MTU 9000/9014)
- [ ] Jumbo frames enabled on Windows (9000/9014)
- [ ] Network offloading enabled on Windows
- [ ] Power management disabled on network adapter
- [ ] SMB extras empty/minimal on UnRaid
- [ ] Switch supports jumbo frames
- [ ] Both systems rebooted after changes

---

## 🎯 The Bottom Line

> **Sometimes less is more.** The "optimization" settings found in many guides actually hurt performance by fighting against the kernel's smart auto-tuning. 
>
> **Jumbo frames + default configuration = optimal performance.**

### Achieved Speeds:

```
Before: 30 MB/s writes, 80 MB/s reads
After:  800 MB/s writes, 222 MB/s reads
Improvement: ~25x faster! 🚀
```

---

## 🖥️ Hardware Tested

- **Server:** UnRaid with Intel i7-8700K
- **Network:** 10 Gigabit Ethernet
- **Switch:** QNAP QSW-2104-2T
- **Client:** Windows 11
- **Cache:** SATA SSD

> *Your mileage may vary based on hardware, but the principles remain the same!*

---

## 📝 Notes

- Guide created through extensive troubleshooting session - December 2025
- The real culprit was `socket options` buffer settings disabling Linux kernel auto-tuning
- SMB signing was never the problem!

---

## 🤝 Contributing

Found this guide helpful? Please consider:
- ⭐ Starring this repository
- 🐛 Opening an issue if you find problems
- 🔀 Submitting a pull request with improvements

---

<div align="center">

### 💖 Support This Project

Found this useful? Want to say thanks and fuel future creations?

**Your support keeps the creativity flowing!** 🍺✨

<table>
  <tr>
    <td align="center">
      <a href="https://www.buymeacoffee.com/3ative">
        <img src="https://img.shields.io/badge/Buy%20Me%20A%20Coffee-Support-yellow.svg?style=for-the-badge&logo=buy-me-a-coffee&logoColor=white" alt="Buy Me A Coffee"/>
        <br/>
        <b>☕ Buy me a Coffee</b>
      </a>
    </td>
    <td align="center">
      <a href="https://www.patreon.com/3ative">
        <img src="https://img.shields.io/badge/Patreon-Become%20a%20Patron-red.svg?style=for-the-badge&logo=patreon&logoColor=white" alt="Patreon"/>
        <br/>
        <b>💖 Join on Patreon</b>
      </a>
    </td>
  </tr>
</table>

**Every contribution helps bring more awesome projects to life!** 🚀

</div>

---

## 📄 License

This guide is released under [MIT License](LICENSE) - feel free to use and share!

---

**Happy optimizing!** 🎉
