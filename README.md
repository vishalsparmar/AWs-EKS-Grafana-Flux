Required GitLab variables
Set these in GitLab → CI/CD → Variables:
TFC_TOKEN        = <Terraform Cloud API token>
TFC_ORG          = <your org name>
TFC_WORKSPACE    = <your workspace name>
🧠 What this solves
✅ If initial run fails → pipeline waits
✅ If you rerun in TFC UI → picks latest run automatically
✅ GitLab reflects final outcome, not first attempt
⚙️ Optional improvement (recommended)
Add timeout protection
