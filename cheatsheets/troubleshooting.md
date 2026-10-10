# Troubleshooting / Tool Fixes

Common "tool won't run" errors and the fix.

---

## Hashcat — no OpenCL/CUDA platform

**Error:**
```
clGetPlatformIDs(): CL_PLATFORM_NOT_FOUND_KHR
ATTENTION! No OpenCL, HIP or CUDA compatible platform found.
```

**Cause:** no OpenCL runtime installed (common in VMs with no GPU).

**Fix (CPU cracking via PoCL):**
```sh
sudo apt install -y pocl-opencl-icd ocl-icd-libopencl1 opencl-headers clinfo
clinfo | grep "Platform Name"   # verify a platform now shows
```

Alternatives:
- `hashcat --force` to push past the warning (slower, may give bad results)
- Crack on a host with a real GPU instead of the VM

---

## NetExec — Neo4j / BloodHound connection errors

**Cause:** NetExec's BloodHound integration can't reach Neo4j — it's not running, or the creds in the config are wrong.

**Fix:**
```sh
# Start Neo4j
sudo neo4j start           # or: sudo systemctl start neo4j

# Check / edit the BloodHound creds + URI
nano ~/.nxc/nxc.conf
```

In `~/.nxc/nxc.conf`, match the `[BloodHound]` section to your Neo4j (default port `7687`):
```ini
[BloodHound]
bh_enabled = True
bh_uri = 127.0.0.1
bh_port = 7687
bh_user = neo4j
bh_pass = <your-neo4j-password>
```

Verify Neo4j is up at `http://localhost:7474` (login = `neo4j` / your password).
