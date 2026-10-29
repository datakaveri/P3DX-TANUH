# P3DX-TANUH

You've built a valuable AI model and you want it run or evaluated on someone else's datasets and evaluation scripts and gauge it’s metrics against other models. But you don't want anyone — not even the people operating those servers — to ever see it. This system makes that possible by passing your model through a series of sealed, tamper-proof vaults, and here's how the whole journey of benchmarking unfolds.

It begins when users come and register on the platform. From there, they reach an encryption package which is accessed through that same interface. Before anything sensitive moves, this encryption package and the first secure vault, called the Buffer TEE, establish a private encrypted connection through a TLS handshake — essentially confirming they're speaking on a line no eavesdropper can read. (A "TEE," or Trusted Execution Environment, is a sealed compartment inside a computer that even the machine's owner cannot peer into.) But a private line isn't enough; you also need proof that the vault is genuine and hasn't been tampered with. So the Buffer TEE sends an attestation report — a digital certificate of authenticity — to an independent referee, the Google Verifier Service, which inspects it and sends back a signed token (a JWT) vouching for it. That token, along with a public key, makes its way back to the encryption package, and once everything checks out, a stronger trusted connection is locked in. Now you can be confident you're handing your model to a real, untampered vault.

With that trust in place, you encrypt your model using the public key and upload it, so it travels as scrambled, unreadable data straight into the Buffer TEE, where it waits. A job scheduler queues up your task, and when its turn comes, the Buffer TEE decrypts the model's weights using its own private key — but only inside the sealed vault, where no human can watch. Based on the specifics of that job, a second vault, the Processing TEE, is spun up to actually do the work, and the same trust ritual repeats: the two vaults shake hands, the Processing TEE proves its authenticity to the Google Verifier Service and gets its own token back, a secure connection is verified, and your model is passed along to it.

This Processing TEE is a fully equipped workspace — it even has the uv Python package manager enabled inside it, so it can pull in and manage the software dependencies the job needs. Just as importantly, it operates under a strict policy. That policy blocks every outward call the vault might try to make and permits only the specific ones explicitly listed — and anyone can confirm the policy in force by checking its policy hash, a unique fingerprint that proves the rules haven't been quietly altered. In other words, your model isn't just sealed in; the vault itself is on a tight leash, unable to phone out anywhere it isn't supposed to.

Now the real work happens. The Processing TEE pulls in the right evaluator scripts based on the dataset chosen, and the encrypted data it needs is fetched in as well. To unlock that data, the vault fetches an identity token and hands it to a Secrets Manager — a gatekeeper that, once it verifies the token, releases the key needed to open the data. With the model, the scripts, and the data all assembled safely inside the sealed environment, the evaluation runs, and the results are gathered up. Finally, those results are populated onto a leaderboard, where you can filter the rankings across various metrics to see how things stack up. From start to finish, your model stays locked away from prying eyes — encrypted in transit, opened only inside vaults that have proven themselves trustworthy, kept on a leash by an auditable policy, and never exposed to the humans running the machines.


## Repositories

The TANUH repositories are linked here as git submodules. Clone with `git clone --recurse-submodules`, or run `git submodule update --init` after cloning.

| Folder | Repository |
|---|---|
| [`TANUH-Buffer-TEE`](https://github.com/datakaveri/TANUH-Buffer-TEE) | RA-TLS submission enclave that receives encrypted models and dispatches benchmark jobs to the Processing TEE |
| [`TANUH-Processing-TEE`](https://github.com/datakaveri/TANUH-Processing-TEE) | Confidential Space workload that decrypts datasets, runs models, adaptors and bucket evaluators, and submits results to the leaderboard |
| [`p3dx-tanuh-buffertee-python`](https://github.com/datakaveri/p3dx-tanuh-buffertee-python) | Python Buffer TEE |
| [`p3dx-tanuh-processingtee-python`](https://github.com/datakaveri/p3dx-tanuh-processingtee-python) | Python Processing TEE |
| [`ui-dx`](https://github.com/prathmeshj1729/ui-dx/tree/feat/faster-dataset-upload) | TANUH web UI, branch `feat/faster-dataset-upload` (see below) |
| [`files-connect-api`](https://github.com/prathmeshj1729/files-connect-api/tree/feat/faster-dataset-upload) | File server, branch `feat/faster-dataset-upload` (see below) |

## Faster dataset upload (not merged)

`ui-dx` and `files-connect-api` point to the branch `feat/faster-dataset-upload` on forks of [`datakaveri/ui-dx`](https://github.com/datakaveri/ui-dx) (private: needs ui-dx access) and [`datakaveri/files-connect-api`](https://github.com/datakaveri/files-connect-api). These changes are proposed, not merged into either repository:

- several files upload at once, without a file list refresh after each one;
- small files are initiated in batches as direct single-PUT uploads (new `POST /databanks/:id/uploads/batch`, `direct` on initiate);
- encryption uses one key per upload session and one manifest per session, with no plaintext hash; each file starts with a header holding its base IV, incremented per chunk;
- large files upload 3 encrypted parts at a time;
- previewing an unknown file type no longer reads the whole object into memory.

To see what changed:

- `ui-dx`: commit [`0f8ade45`](https://github.com/prathmeshj1729/ui-dx/commit/0f8ade45853b7583561da45a7e5f820615e2cea1). The commit before it only runs Prettier on the touched files.
- `files-connect-api`: [compare with `tanuh-gcp/v2.3`](https://github.com/datakaveri/files-connect-api/compare/tanuh-gcp/v2.3...prathmeshj1729:files-connect-api:feat/faster-dataset-upload).

The Processing TEE does not read the new encrypted format yet, and the UI needs the file-server change deployed first.
