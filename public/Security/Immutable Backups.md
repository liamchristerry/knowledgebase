
Tags: #veeam #immutablity #security #backup 
# Veeam Immutability and WORM

**References:**
- [How Immutability Works (Object Storage) - Veeam B&R 13](https://helpcenter.veeam.com/docs/vbr/userguide/hiw_immutability_os.html?ver=13)
- [How Immutability Works (Hardened Repository) - Veeam B&R 13](https://helpcenter.veeam.com/docs/vbr/userguide/hardened_repository_immutability.html?ver=13)

---

## WORM Is Not One Thing

WORM (Write Once Read Many) describes a *behavior*, not a specific technology.
How that behavior is enforced — and how trustworthy it is — varies significantly
by implementation.

```
Strongest 
 
Object Lock — Compliance Mode 
		│ 
	Deletion is cryptographically impossible until lock expires. 
		│ 
	No administrative override exists, including the root account. 
	

Hardened Linux Repository (inode immutability) 
		│ 
	Deletion is blocked at the filesystem level. 
		│ 
	Requires root to override — root access is intentionally withheld from Veeam. 
	

Appliance WORM (ExaGrid, StoreOnce, etc.) 
		│ 
	Deletion is blocked by vendor policy. 
		│ 
	Admin-level overrides typically exist within the appliance. 
	

Weakest
```


---

## Object Storage WORM vs. Appliance WORM

These are both marketed as WORM. The difference is *where* enforcement lives.

| | Azure Blob / S3 Object Lock | ExaGrid / StoreOnce |
|---|---|---|
| Type | Storage-native | Appliance / policy-based |
| Enforcement | Part of the storage service itself | Implemented by the vendor product |
| Delete during lock | Not a valid operation — rejected at the API layer | Blocked by policy |
| Administrative override | None (Compliance mode) | Exists at the admin level |
| Trust model | Protected by time, not trust | Protected by policy and access control |

Both are WORM. Only one is truly override-proof.

---

## How Object Locking Works

Object lock protection is **time-based, not permission-based**. When Veeam writes
a backup object to an object storage repository, the storage system applies a lock
with an expiry timestamp to that specific object. Until that timestamp passes, the
storage layer rejects any delete or overwrite request — regardless of what API
credentials are used or what permissions they carry.


```
Veeam writes backup object to S3 
				| 
▼ Storage applies WORM lock + retention expiry to the object 
				| 
▼ Lock is enforced at the storage data plane Even a root/admin API key cannot delete it until expiry
```

### API Key Permissions Are Defense in Depth, Not the Lock

In a hardened Veeam setup, the S3 credentials given to Veeam are scoped to read
and write only — no delete permission. This is a second, independent layer:

- Veeam writes new backup objects and reads them for restore — it never needs to
  delete objects directly.
- Retention cleanup is handled by the lock expiring naturally, or by a bucket
  lifecycle policy, not by Veeam issuing delete calls.
- If the Veeam server is compromised and the API key is stolen, the attacker
  cannot delete objects because: (1) the key has no delete permission, and (2)
  the objects are WORM locked regardless.

Neither layer alone is sufficient. Together they are.

### Compliance Mode vs. Governance Mode

**Governance mode** — The lock can be bypassed by an IAM principal with the
`s3:BypassGovernanceRetention` permission. Useful when operational flexibility
is needed (e.g., correcting a mistake).

**Compliance mode** — No override exists. Not the root account, not the cloud
provider, not anyone. The object cannot be deleted or have its retention shortened
until the lock expires. This is the appropriate mode for ransomware-hardened
backup storage.

> [!note] Veeam and minimum immutability periods
> Veeam does not support custom object lock configurations on repositories used
> as capacity tier extents. If a bucket is added with a non-standard lock
> configuration, Veeam will fall back to the minimum immutability period.

### Supported Object Lock Implementations (Veeam B&R 13)

- **Object Lock + Versioning** — Amazon S3, S3-compatible, IBM Cloud, Wasabi,
  11:11 Cloud Object Storage, Google Cloud Storage
- **Version-level WORM + Blob Versioning** — Azure Blob Storage

---

## Hardened Linux Repository

Veeam's on-premises immutability option uses Linux filesystem-level immutability
rather than object lock. The mechanism is different but the goal is the same.

### How It Works

Linux filesystems support an immutable flag on individual files, set via the
`chattr +i` command. When set, the file cannot be modified, renamed, or deleted
by any user — including root — while the flag is active. Veeam sets this flag
on backup files after they are written.

```
Veeam writes backup file to hardened repo | ▼ Veeam sets chattr +i (immutable flag) on the file | ▼ File is now immutable at the inode level Even root cannot delete it while the flag is set
```

### Access Model

The Linux account Veeam uses to access the hardened repository is intentionally
not root and has no sudo rights. It has access only to the backup storage path.
This means:

- A compromised Veeam server cannot elevate to root on the repository OS.
- A compromised Veeam service account cannot call `chattr -i` to remove the flag.
- Removing immutability requires direct root access to the Linux host itself.

### Single-Use Credentials

The Single Use Credentials is the way Veeam can access the Linux server repository, the Veeam Admin enters in the username and password. Veeam connects and then uses SSH keys to future connections

https://helpcenter.veeam.com/archive/backup/110/vsphere/linux_server_ssh.html

