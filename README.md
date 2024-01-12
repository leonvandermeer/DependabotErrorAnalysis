# This repository reproduces an issue where Dependabot fails to update a NuGet dependency

## Summary

When a project in my repository refers to a NuGet package, Dependabot correctly detects that this package needs an update.

However, when my repository also contains a project that has a project reference to the project referencing the NuGet package (and thus transitively refers to the NuGet package), Dependabot fails.

IMHO, this is a bug in Dependabot. It should update the project containing the PackageReference. It should not error out on the project containing the ProjectReference.

## This repository's contents show an example of this bug

Following diagram shows the dependency graph of the 2 projects and the NuGet package:
![Dependency Diagram](http://yuml.me/diagram/scruffy%3bdir:td/class/[VOnMyRoute]ProjectReference->[Updates.Updates],[Updates.Updates]PackageReference->[Octokit] "Dependency Diagram")

Follow along with [Dependabot's log](https://github.com/leonvandermeer/DependabotErrorAnalysis/network/updates/784277825) (I ommitted the cluttering ```proxy``` messages):

```
updater | 2024-02-07T18:21:55.096362409 [784277825:main:WARN:src/devices/src/legacy/serial.rs:222] Detached the serial input due to peer close/error.
updater | time="2024-02-07T18:21:58Z" level=info msg="guest starting" commit=409d83fb821a7c266460959144006f8ddc985a54
updater | time="2024-02-07T18:21:58Z" level=info msg="starting job..." fetcher_timeout=10m0s job_id=784277825 updater_timeout=45m0s updater_version=3f3794dbabfb2ac80748106525768068016365a0-nuget
updater | 2024/02/07 18:22:04 INFO <job_784277825> Starting job processing
updater | 2024/02/07 18:22:05 INFO <job_784277825> Finished job processing
updater | time="2024-02-07T18:22:05Z" level=info msg="task complete" container_id=job-784277825-file-fetcher exit_code=0 job_id=784277825 step=fetcher
updater | 2024/02/07 18:22:10 INFO <job_784277825> Starting job processing
updater | 2024/02/07 18:22:11 INFO <job_784277825> Starting update job for leonvandermeer/DependabotErrorAnalysis
updater | 2024/02/07 18:22:11 INFO <job_784277825> Checking all dependencies for version updates...
updater | 2024/02/07 18:22:11 INFO <job_784277825> Checking if Octokit 9.1.1 needs updating
updater | running NuGet updater:
updater | /opt/nuget/NuGetUpdater/NuGetUpdater.Cli framework-check --project-tfms net8.0 --package-tfms .NETStandard2.0 --verbose
updater | The package is compatible.
updater | 2024/02/07 18:22:13 INFO <job_784277825> Latest version is 9.1.2
updater | 2024/02/07 18:22:13 INFO <job_784277825> Requirements to unlock all
updater | 2024/02/07 18:22:13 INFO <job_784277825> Requirements update strategy 
updater | Finding updated dependencies for Octokit.
```

Dependabot has detected that Octokit[^1] NuGet package reference needs to be updated and starts doing so:
[^1]: Great library!

```
updater | 2024/02/07 18:22:14 INFO <job_784277825> Updating Octokit from 9.1.1 to 9.1.2
updater | running NuGet updater:
```

Dependabot runs the NuGet updater. Looking at some [source code](https://github.com/dependabot/dependabot-core/blob/3aa1e0cfdb71e46ab33915830e46f078b6bbfed0/nuget/lib/dependabot/nuget/file_updater.rb#L62), it seems that the NuGet updater runs NuGetUpdater.Cli for each project. It starts with **VOnMyRoute/VOnMyRoute.csproj**:

```
updater | /opt/nuget/NuGetUpdater/NuGetUpdater.Cli update --repo-root /home/dependabot/dependabot-updater/repo --solution-or-project /home/dependabot/dependabot-updater/repo/VOnMyRoute/VOnMyRoute.csproj --dependency Octokit --new-version 9.1.2 --previous-version 9.1.1 --verbose
updater |   No dotnet-tools.json files found.
updater | Running for project [/home/dependabot/dependabot-updater/repo/VOnMyRoute/VOnMyRoute.csproj]
updater |   Running for SDK-style project
updater |     Package [Octokit] Does not exist as a dependency in [/home/dependabot/dependabot-updater/repo/VOnMyRoute/VOnMyRoute.csproj].
```

True, VOnMyRoute.csproj does not refer to Octokit directly[^2].

[^2]: Dependabot's source code where this is logged can be found [here](https://github.com/dependabot/dependabot-core/blob/8f858a0f3e2943644a0deb6c07bb78a958346085/nuget/helpers/lib/NuGetUpdater/NuGetUpdater.Core/Updater/SdkPackageUpdater.cs#L55).

```
updater | Update complete.
```

Hmm, what about the **Updates.Updates.csproj** project? IMHO, it is a bug that it is skipped!

```
updater | 2024/02/07 18:22:20 INFO <job_784277825> [Transport] Sending envelope with items [event] c9dd394e726e444cb623e8874a254ba4 to Sentry
updater | 2024/02/07 18:22:20 ERROR <job_784277825> Error processing Octokit (Dependabot::DependabotError)
updater | 2024/02/07 18:22:20 ERROR <job_784277825> FileUpdater failed
```

Dependabot detects that not any file is modified (```updated_files``` is empty) and [this source code](https://github.com/dependabot/dependabot-core/blob/8f858a0f3e2943644a0deb6c07bb78a958346085/updater/lib/dependabot/dependency_change_builder.rb#L43)) throws.

```
updater | 2024/02/07 18:22:20 ERROR <job_784277825> /home/dependabot/dependabot-updater/lib/dependabot/dependency_change_builder.rb:43:in `run'
updater | 2024/02/07 18:22:20 ERROR <job_784277825> /home/dependabot/dependabot-updater/lib/dependabot/dependency_change_builder.rb:26:in `create_from'
updater | 2024/02/07 18:22:20 ERROR <job_784277825> /home/dependabot/dependabot-updater/lib/dependabot/updater/operations/update_all_versions.rb:128:in `check_and_create_pull_request'
updater | 2024/02/07 18:22:20 ERROR <job_784277825> /home/dependabot/dependabot-updater/lib/dependabot/updater/operations/update_all_versions.rb:60:in `check_and_create_pr_with_error_handling'
updater | 2024/02/07 18:22:20 ERROR <job_784277825> /home/dependabot/dependabot-updater/lib/dependabot/updater/operations/update_all_versions.rb:35:in `block in perform'
updater | 2024/02/07 18:22:20 ERROR <job_784277825> /home/dependabot/dependabot-updater/lib/dependabot/updater/operations/update_all_versions.rb:35:in `each'
updater | 2024/02/07 18:22:20 ERROR <job_784277825> /home/dependabot/dependabot-updater/lib/dependabot/updater/operations/update_all_versions.rb:35:in `perform'
updater | 2024/02/07 18:22:20 ERROR <job_784277825> /home/dependabot/dependabot-updater/lib/dependabot/updater.rb:45:in `run'
updater | 2024/02/07 18:22:20 ERROR <job_784277825> /home/dependabot/dependabot-updater/lib/dependabot/update_files_command.rb:43:in `perform_job'
updater | 2024/02/07 18:22:20 ERROR <job_784277825> /home/dependabot/dependabot-updater/lib/dependabot/base_command.rb:36:in `run'
updater | 2024/02/07 18:22:20 ERROR <job_784277825> bin/update_files.rb:24:in `<main>'
updater | 2024/02/07 18:22:20 INFO <job_784277825> Finished job processing
updater | 2024/02/07 18:22:20 INFO Results:
updater | Dependabot encountered '1' error(s) during execution, please check the logs for more details.
updater | +-------------------------------+
updater | | Dependencies failed to update |
updater | +---------------+---------------+
updater | | Octokit       | unknown_error |
updater | +---------------+---------------+
updater | time="2024-02-07T18:22:21Z" level=info msg="task complete" container_id=job-784277825-updater exit_code=0 job_id=784277825 step=updater
```