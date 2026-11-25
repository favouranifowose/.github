# Security

Do not post credentials or vulnerability details in public issues. Use **Security → Report a vulnerability** on the affected repository when private vulnerability reporting is enabled. Otherwise, contact [Favour Anifowose](https://github.com/favorstic) through a private contact channel listed on that profile; share sensitive details only after establishing a private channel.

The latest `v1` release receives security fixes. Callers should use the maintained major tag and review workflow updates. Private repositories use GitHub Free without environment approval gates: agents may push directly to `main`, and pipelines deploy from `main` after CI passes. Keep deployment credentials narrowly scoped and stored in repository secrets. Report a leaked credential privately and revoke it promptly in the issuing service.
