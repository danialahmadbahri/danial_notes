# FocusGuard — FG-001 Engineering Notes & Portfolio Evidence

**Organisation:** Focus Group (fictional UK AI consultancy)  
**Project:** FocusGuard — AWS Governance & Cost Automation  
**Ticket:** FG-001 — AWS Identity Inspector  
**Status:** AWS identity foundation configured; local SSO authentication and Python STS integration pending verification  
**Last updated:** 19 November 2026

> This document records the engineering work performed, why the design was chosen, what to revise for interviews, and the evidence still needed. It intentionally contains no AWS account numbers, credentials, portal URLs, or personal information.

## 1. Problem and goal

Cloud automation must establish **which AWS identity and account it is using** before inspecting or changing infrastructure. FG-001 builds a small Python tool that obtains its identity through AWS Security Token Service (STS) using Boto3, reports the account ID and assumed-role ARN safely, and handles authentication errors clearly.

**Acceptance criteria (pending):**
- AWS CLI authenticates through IAM Identity Center using a named profile and temporary credentials.
- `aws sts get-caller-identity --profile focusguard-dev` succeeds.
- Python/Boto3 retrieves the same identity with the chosen profile.
- Missing or expired authentication is handled gracefully.
- No root access keys, hard-coded secrets, or chargeable application infrastructure are required.

## 2. Verified progress and outstanding work

| Area | Evidence/status |
|---|---|
| macOS development environment | AWS CLI v2.39.0, Python 3.15.2 (observed) |
| Python dependency | Boto3 1.44.4 installed in project virtual environment (observed) |
| Project workspace | `~/Projects/focus-group/focusguard` with `src/`, `tests/`, `reports/`, `docs/` |
| First Python execution | `src/main.py` printed `FocusGuard — AWS Identity Inspector` |
| Initial CLI configuration | `aws configure list-profiles` showed no profiles; `aws configure list` showed no configured credentials or Region |
| Root account | Root MFA confirmed enabled; **root should not be used for daily engineering** |
| AWS Organizations | Organization created; existing account became management account |
| IAM Identity Center | Organization instance enabled, single Region |
| Workforce identity | `focusgroup-engineer` created |
| Permission set | `FocusGuard-Identity-Inspector` created with inline STS identity-check policy; one-hour session duration |
| Account assignment | Engineering user assigned to existing AWS account with that permission set |
| AWS CLI SSO profile | **Not yet verified** |
| Successful STS API response | **Not yet verified** |
| Boto3 STS integration | **Not yet implemented/verified** |

Do not claim completed SSO login or a functioning Boto3 identity inspector until terminal evidence confirms them.

## 3. Architecture

```text
Developer on Mac
    |
    | aws sso login --profile focusguard-dev
    v
AWS IAM Identity Center (organization instance)
    |
    | account assignment + permission set
    v
Temporary assumed-role credentials
    |
    +--> AWS CLI --> STS GetCallerIdentity
    |
    +--> Python/Boto3 --> STS GetCallerIdentity
                               |
                               v
                       Account, UserId, ARN
```

The CLI and Boto3 can use the same named AWS configuration profile. The SSO session is separate from the Python virtual environment.

## 4. Concepts to memorise and explain

| Term | Plain-English explanation | Why it matters |
|---|---|---|
| AWS account | Security and billing boundary for AWS resources | Prevent accidental operations in the wrong account |
| Root user | Account owner identity with full privileges | Reserve for tasks that require root; protect with MFA |
| MFA | Extra authentication factor | Reduces risk from stolen passwords |
| AWS Organizations | Central service to organize and govern AWS accounts | Enables organization-wide identity and future account structure |
| Management account | Account that administers an AWS Organization | Sensitive; minimize routine use and permissions |
| IAM Identity Center | Central workforce sign-in and AWS account access | Supports temporary CLI credentials without IAM user keys |
| Organization instance | Identity Center deployment supporting AWS account assignments across an organization | Correct fit for CLI SSO access to AWS accounts |
| Account instance | Identity Center deployment for supported applications within one account | Does not replace organization-instance AWS account access |
| Permission set | Template for AWS permissions assigned to users/groups for an account | Defines what an authenticated user may do |
| Account assignment | Links a user/group, permission set, and AWS account | A permission set alone does not grant account access |
| IAM policy | JSON document describing permissions | Enables least-privilege design |
| Least privilege | Give only permissions needed for a task | Limits impact of mistakes or compromise |
| AWS STS | Security Token Service | Provides temporary credentials and caller-identity API |
| `GetCallerIdentity` | STS API returning account, principal ID, and ARN | Confirms the identity currently used by automation |
| ARN | Amazon Resource Name identifying an AWS principal/resource | Helps audit which assumed role was used |
| AWS CLI | Terminal interface for AWS APIs | Useful for verification and troubleshooting |
| SDK | Software Development Kit | Lets application code call AWS APIs |
| Boto3 | AWS SDK for Python | Lets FocusGuard call STS from Python |
| Python virtual environment | Isolated local Python dependencies | Keeps project packages separate from system Python |
| AWS profile | Named local CLI/SDK configuration | Avoids ambiguity when using multiple accounts/roles |
| AWS Region | Geographic deployment location for many AWS services | Identity Center's home Region and CLI default Region can differ |

### Key distinction: authentication vs authorization

- **Authentication:** proving who you are (Identity Center sign-in).
- **Authorization:** deciding what that identity may do (permission set + account assignment).
- **Assume role:** obtaining a session for a role with the permissions assigned to it.

### Key distinction: single Region vs multi-Region

A single-Region Identity Center setup keeps the identity service configuration in one Region. It **does not** restrict AWS application resources to that Region. Multi-Region identity deployment adds complexity and was not required for this initial portfolio environment.

## 5. Permission policy and its nuance

The custom permission set was configured with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "IdentifyCaller",
      "Effect": "Allow",
      "Action": "sts:GetCallerIdentity",
      "Resource": "*"
    }
  ]
}
```

- `Version`: IAM policy language version.
- `Statement`: collection of permission rules.
- `Sid`: optional statement identifier.
- `Effect`: allow or deny.
- `Action`: API operation covered by the statement.
- `Resource: "*"`: STS GetCallerIdentity does not support resource-level permission scoping.

**Important nuance:** AWS does not require an explicit IAM Allow for `sts:GetCallerIdentity`; even an explicit Deny does not prevent the API from returning caller identity. The policy documents the intended minimal use, rather than being a security boundary for that API. The role has no general read/write permissions merely because this statement exists. `ReadOnlyAccess` was rejected because it grants much broader visibility across AWS services.

## 6. Commands to know

| Command | Reminder |
|---|---|
| `cd ~/Projects/focus-group/focusguard` | Enter project directory |
| `source .venv/bin/activate` | Activate Python virtual environment |
| `python -m pip show boto3` | Check installed Boto3 |
| `aws --version` | Check AWS CLI installation |
| `aws configure list-profiles` | Show configured profile names |
| `aws configure list` | Inspect selected AWS configuration sources; redact values before sharing |
| `aws configure sso` | Create a CLI SSO profile |
| `aws sso login --profile focusguard-dev` | Authenticate the named profile through the browser |
| `aws sts get-caller-identity --profile focusguard-dev` | Confirm caller identity through AWS STS |
| `aws configure list --profile focusguard-dev` | Inspect profile resolution and credential source |
| `aws sso logout` | Clear cached SSO sign-in sessions |
| `deactivate` | Leave Python virtual environment |

**Never paste access keys, session tokens, login codes, passwords, or contents of `~/.aws/credentials` into GitHub or project notes.** Redact account identifiers and ARN details in public screenshots/logs where appropriate.

## 7. Next practical engineering milestones

1. In IAM Identity Center, locate the AWS access portal URL and note the Identity Center home Region **privately**.
2. Run `aws configure sso`; use session name `focusgroup`, select the existing account and `FocusGuard-Identity-Inspector`, and save the profile as `focusguard-dev`.
3. Run `aws sso login --profile focusguard-dev` and complete browser authentication as the engineering user.
4. Run `aws sts get-caller-identity --profile focusguard-dev` and verify the returned account and assumed-role ARN.
5. Build `src/main.py` to create a Boto3 session using `profile_name="focusguard-dev"`, call `sts.get_caller_identity()`, and print safe, useful output.
6. Add error handling for absent profile, expired login, and other AWS authentication failures; test success and failure paths.
7. Add tests and a sanitized example report, then commit evidence to GitHub.

**Troubleshooting tip:** If CLI login succeeds but Python fails, check the named profile, Boto3 session construction, cached SSO session, and exception message. Do not fix this by creating root or IAM-user access keys.

## 8. Git and GitHub safety

Recommended repository layout:

```text
focus-group/
  .gitignore
  README.md
  focusguard/
    src/main.py
    docs/FOCUSGUARD_ENGINEERING_NOTES.md
    tests/
    reports/
```

Recommended `.gitignore`:

```gitignore
# Python environments and generated files
.venv/
__pycache__/
*.py[cod]
.pytest_cache/

# Environment and local secrets
.env
.env.*
!.env.example
*.pem
*.key

# Local AWS configuration, if copied into the project
.aws/

# macOS / editor
.DS_Store
.vscode/
```

**Before every commit:** inspect `git status`, `git diff --cached`, and staged filenames. Never add `.venv`, AWS credentials, access portal URLs, private keys, tokens, or unredacted account data. `.gitignore` only prevents untracked files from being added; it does not untrack files already committed.

## 9. Portfolio story / interview talking points

> I began FocusGuard by establishing secure developer authentication before building AWS automation. I enabled an AWS Organization and an IAM Identity Center organization instance, created a dedicated engineering identity, and assigned a minimal custom permission set to my existing AWS account. I deliberately avoided root access keys and broad read-only permissions. My next milestone is to validate temporary CLI SSO credentials and use Boto3 to call AWS STS GetCallerIdentity, with explicit error handling and tests.

**Be prepared to explain:**
- Why a root account with MFA is still not suitable for routine automation.
- Why a permission set needs an AWS account assignment.
- Why temporary credentials are preferable to long-lived keys.
- Why `ReadOnlyAccess` was excessive for an identity-only task.
- What `Account`, `UserId`, and `Arn` reveal about the current caller.
- How to verify a command succeeded using real terminal evidence rather than assumptions.
- Why a `.gitignore` and a staged-diff review matter before pushing to a public repository.

## 10. Evidence to add later

- Sanitized CLI SSO profile configuration evidence (without tokens, account ID, or portal URL).
- Sanitized successful STS response.
- Python code and tests demonstrating success and authentication failure handling.
- Short architecture diagram and design decision record.
- Example of a failure investigated and fixed, including the lesson learned.

**Portfolio integrity:** This document distinguishes completed AWS setup from pending authentication and Python implementation. Update the status only after testing each milestone.
