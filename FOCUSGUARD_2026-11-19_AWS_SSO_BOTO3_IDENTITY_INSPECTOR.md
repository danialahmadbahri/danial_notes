# FocusGuard FG-001 — From AWS sign-in to a working Python tool

19 November 2026 · Engineering journal

This entry follows my [original FocusGuard engineering notes](FOCUSGUARD_ENGINEERING_NOTES.md). FocusGuard is an AWS governance and automation project. Its first feature checks which AWS account and identity a script is using before it does any further work.

## What I built

I configured an AWS CLI profile named `focusguard-dev` for IAM Identity Center and signed in as my engineering user. I verified the assigned role with AWS Security Token Service (STS), then used Boto3, the AWS SDK for Python, to make the same identity request from `src/main.py`.

Both checks returned the expected account and assumed role. The Python request returned HTTP 200 with zero retries. It only read identity information.

## The commands I used

| Command | What it does |
| --- | --- |
| `aws configure sso` | Saves the IAM Identity Center settings as a named AWS CLI profile. |
| `aws configure list-profiles` | Lists saved profile names so I can confirm the configuration exists. |
| `aws sso login --profile focusguard-dev` | Starts browser sign-in for the saved profile. |
| `aws sts get-caller-identity --profile focusguard-dev` | Asks STS which AWS account and identity the profile is using. |
| `python src/main.py` | Runs the Python inspector from the active project environment. |
| `cat -n src/main.py` | Displays the source with line numbers for review. |

## What went wrong and how I fixed it

My first sign-in failed because the CLI couldn't find `focusguard-dev`. During setup, I also entered a command at the prompt for the SSO session name and cancelled that attempt. I reran the setup with `focusgroup` as the session name, selected my assigned account and role, and saved the profile as `focusguard-dev`. I confirmed the profile existed before signing in again.

The two names serve different purposes. `focusgroup` names the SSO session configuration. `focusguard-dev` is the profile selected with `--profile` in the CLI and `profile_name` in Boto3.

## How I improved the Python output

The first version printed the full AWS response, including HTTP headers. I used Python dictionary keys to display just `Account`, `UserId` and `Arn`. I also spotted a repeated heading in the terminal output. A numbered source review confirmed the latest file contained one heading statement.

## Next step

Test how the script behaves with a nonexistent local profile, then handle the resulting `ProfileNotFound` exception with a message that tells the user how to recover.
