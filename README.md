# First hands-on of running UI tests on M1-based macOS with CircleCI

For both CircleCI-managed macOS executors (using simulators) and self-hosted runners (using either simulators or physical devices connected to runner nodes).

## Prerequisites

### Clone this repository

Clone the entire repository to your own organization to work on the stuff.

Important: Clone this repository **_to the organization a resource class will be created later (explained below)_**. Runner resource classes are accessible only from projects under the same organization.

### Create a resource class

Important: **This step needs to be conducted only once by _an organization admin_**!

1.  Go to https://app.circleci.com/runners/github_OR_bitbucket/YOUR_ORG_NAME/inventory (modify `github_OR_bitbucket` and `YOUR_ORG_NAME` accordingly - bear in mind they are case sensitive!).
2.  Hit the button labelled "Create Resource Class" at the upper-right corner.
3.  Fill in the form. **Important: This action can be done only once. Be mindful on what you input. Especially if you are to create a new namespace.**
4.  Hit the button labelled "Save and Continue" under the form.
5.  In the next page you see a token required to set up a runner node. Keep it in a secure and accessible place - we use the token later.

Note: We will reuse the token generated herein for convenience, but tokens can be newly created or revoked at anytime.

### Have your Mac environment handy

Going forward we will arm your Mac environment as a runner node. That means, we need a Mac.

### Set up functional Xcode

The pipeline is extensively relying on Xcode, specifically `simctl` and `xcodebuild`.
The toolchain needs to be installed before the pipeline is executed.
(It expects that those commands are already available at runtime.)

Below is a convenient snippet for you to install them - assuming you already have [Homebrew](https://brew.sh/) installed, and `/opt/homebrew/bin` and `/opt/homebrew/sbin` are in your `PATH` for both Zsh and Bash (i.e., `/etc/paths.d`):

```
brew install xcodesorg/made/xcodes
xcodes install --select 14.2.0
```

Note:

1.  As shown in the example command the pipeline is tested against Xcode 14.2.0.
2.  The installation process requires an accessible Apple ID.
3.  The installation process requires the system user password for the (privileged) user running the process.
4.  The installation process would take a _long_ time.

See also: https://github.com/XcodesOrg/xcodes#usage

### Pre-install `jq` and `xcpretty`

While it is technically possible to install toolchains at every beginning of each pipeline run, that is awfully inefficient.
Install them beforehand instead.

Below is a convenient snippet for you to install them - assuming you already have [Homebrew](https://brew.sh/) installed, and `/opt/homebrew/bin` and `/opt/homebrew/sbin` are in your `PATH` for both Zsh and Bash (i.e., `/etc/paths.d`):

```
brew install jq
sudo gem install xcpretty
```

Double check that both `jq` and `xcpretty` are in the default `PATH` after installation.

## Arm your Mac as a CircleCI runner node

The instruction below is slightly different from what instructed in CircleCI's documentation, in order to include tricks for avoiding common errors.

### Install CircleCI's launch agent

```
bash -c '
platform=darwin/arm64

set -eu pipefail

echo "Installing CircleCI Runner for ${platform}"

base_url="https://circleci-binary-releases.s3.amazonaws.com/circleci-launch-agent"
if [[ -z ${agent_version+x} ]]; then
    agent_version=$(curl "${base_url}/release.txt")
fi

# Set up the runner directories
echo "Setting up CircleCI Runner directories"
sudo mkdir -p /var/opt/circleci /opt/circleci

# Downloading launch agent
echo "Using CircleCI Launch Agent version ${agent_version}"
echo "Downloading and verifying CircleCI Launch Agent Binary"
curl -sSL "${base_url}/${agent_version}/checksums.txt" -o checksums.txt
file="$(grep -F "${platform}" checksums.txt | cut -d " " -f 2 | sed "s/^.//")"
mkdir -p "${platform}"
echo "Downloading CircleCI Launch Agent: ${file}"
curl --compressed -L "${base_url}/${agent_version}/${file}" -o "${file}"

# Verifying download
echo "Verifying CircleCI Launch Agent download"
if grep "${file}" checksums.txt | shasum -a 256 --check; then
    sudo install -m 0755 "${file}" "/opt/circleci/circleci-launch-agent"
else
    echo "Invalid checksum for CircleCI Launch Agent, please try download again"
fi

sudo mkdir -p /var/opt/circleci
sudo chmod 0777 /var/opt/circleci
'
```

### Configure launch agent

Edit `YOUR_RUNNER_TOKEN_COMES_HERE` below accordingly, then execute the snippet.

```
sudo mkdir -p "/Library/Preferences/com.circleci.runner"
sudo tee /Library/Preferences/com.circleci.runner/launch-agent-config.yaml <<EOD
api:
  auth_token: YOUR_RUNNER_TOKEN_COMES_HERE

runner:
  name: "$(hostname)"
  command_prefix : ["sudo", "-niHu", "$(whoami)", "--"]
  working_directory: /var/opt/circleci/workdir
  cleanup_working_directory: true

logging:
  file: /Library/Logs/com.circleci.runner.log
EOD
sudo chmod 600 /Library/Preferences/com.circleci.runner/launch-agent-config.yaml
```

### Configure `launchd` to run launch agent as a service

```
sudo tee /Library/LaunchDaemons/com.circleci.runner.plist <<EOD
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>Label</key>
        <string>com.circleci.runner</string>

        <key>Program</key>
        <string>/opt/circleci/circleci-launch-agent</string>

        <key>ProgramArguments</key>
        <array>
            <string>circleci-launch-agent</string>
            <string>--config</string>
            <string>/Library/Preferences/com.circleci.runner/launch-agent-config.yaml</string>
        </array>

        <key>RunAtLoad</key>
        <true/>

        <!-- The agent needs to run at all times -->
        <key>KeepAlive</key>
        <true/>

        <!-- This prevents macOS from limiting the resource usage of the agent -->
        <key>ProcessType</key>
        <string>Interactive</string>

        <!-- Increase the frequency of restarting the agent on failure, or post-update -->
        <key>ThrottleInterval</key>
        <integer>3</integer>

        <!-- Wait for 10 minutes for the agent to shut down (the agent itself waits for tasks to complete) -->
        <key>ExitTimeOut</key>
        <integer>600</integer>

        <!-- Create session to allow access to keychains -->
        <key>SessionCreate</key>
        <true/>

        <!-- The agent uses its own logging and rotation to file -->
        <key>StandardOutPath</key>
        <string>/dev/null</string>
        <key>StandardErrorPath</key>
        <string>/dev/null</string>
    </dict>
</plist>
EOD
sudo launchctl load /Library/LaunchDaemons/com.circleci.runner.plist
```

## Run your first pipeline!

1.  Go to https://app.circleci.com/projects/project-dashboard/github_OR_bitbucket/YOUR_ORG_NAME/ (modify `github_OR_bitbucket` and `YOUR_ORG_NAME` accordingly - bear in mind they are case sensitive!)
2.  Find the repository in the list of the project.
3.  Hit the button labelled "Set Up Project".
4.  Choose the "Fastest" option, and type `main` as the branch name.
5.  Hit the button labelled "Set Up Project" in the modal.
6.  You will see that your first pipeline is running! However, it will show an error complaining that the resource class is not permitted. Go to `.circleci/config.yml` and edit the resource class accordingly.
7.  Now you see you are running your job on your own Mac, with your devices enumerated in available devices!

## Further configuration to test the app on your physical device

Tips to actually run a test on a physical device.
In case you have never run a test on your physical devices.

### Make sure to connect your iOS device to your runner node

Make sure also to enable Developer Mode on your iOS device, if your device is running iOS 16 or above; in order to do this you would need to open Xcode with your iOS device connected.

### Prepare 2 profiles

You would need 2 profiles, one for `com.example.your-app`, and the other with a postfix `xctrunner`, i.e., for `com.example.your-app.xctrunner`.
As far as the author knows, test builds will automatically add the suffix to the bundle ID at build time, unless you apply a neat tweak.

Additionally, as long as the author knows, the profile for `com.example.your-app.xctrunner` needs to be linked to a developer certificate (not a distribution certificate).

### Run launch-agent in an interactive shell

There are certain cases where code sign fails if it is executed under `launchd`.
The easiest technique to avoid the traps is to run launch-agent in an interactive shell.

Run the snippet below and it will invoke launch-agent in an interactive in-background shell (with a dedicated TTY):

```
sudo launchctl unload /Library/LaunchDaemons/com.circleci.runner.plist
sudo screen -dm /opt/circleci/circleci-launch-agent --config /Library/Preferences/com.circleci.runner/launch-agent-config.yaml
```

## Acknowledgements

The Xcode project in use (along with the codes written in Swift) in this demo was originally developed by Tadashi Nemoto ([@tadashi0713](https://github.com/tadashi0713)). Original project is available at https://github.com/tadashi0713/circleci-demo-ios. Thanks Tadashi, I was able to build this demo without being exhausted with Xcode which does not run on my main Windows machines!
