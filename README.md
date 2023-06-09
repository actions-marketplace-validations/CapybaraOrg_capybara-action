<p align="center">
  <a href="https://taikai.network/gsf/hackathons/carbonhack22/projects/cl975ox6936793201uh6zkzqkrt/idea">
    <img alt="Gatsby" src="docs/assets/logo.png" width="64" />
  </a>
</p>
<h1 align="center">
  Capybara GitHub Action
</h1>

<p align="center">
  <a href="https://github.com/CapybaraOrg/capybara-action/blob/main/LICENSE">
    <img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="Project is released under the MIT license." />
  </a>
  <a href="https://github.com/prettier/prettier">
    <img src="https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square" alt="code style: prettier" />
  </a>
</p>

# Capybara GitHub Action

Capybara is a GitHub Action used to automatically reschedule workflow runs depending on the most carbon efficient time.

## Carbon Aware WebApi

Capybara makes use of the Carbon Aware WebApi developed by [Green Software Foundation](https://greensoftware.foundation/). For more
information [see GSF repository](https://github.com/Green-Software-Foundation/carbon-aware-sdk).

---

## Table of contents[](#table-of-contents)

1. [Motivation](#motivation)
2. [Composition](#composition)
3. [Usage and Setup](#usage_and_setup)
   - [Inputs](#inputs)
   - [Outputs](#outputs)
   - [Example usage](#example-usage)
4. [Development](#development)
   - [Setup](#setup)
   - [Publish a new version](#publish-new-version)
5. [Future improvements](#future-improvements)
6. [Gotchas](#gotchas)

---

## Motivation[](#motivation)

Every day developers around the world run GitHub Actions workflows that are not necessarily part of a CI/CD development.
Nightly security checks and builds, database backups - those workflows we usually schedule to run at the same time every
day, week, or month.
They have some time constraints but can be triggered after a period of time. For instance, a run that is usually
triggered at midnight will be ok to run anytime between 10 PM and 8 AM (when no one is working on the code). Because of
this flexibility, routinely run workflows are an excellent area for utilising carbon-aware computing.

Carbon-aware computing is the idea that you can reduce the carbon footprint of your application just by running things
at different times or locations. The carbon intensity of energy varies due to the different proportions of renewable vs.
fossil fuel energy sources.
Our Capybara tool, when added to your workflow as a step (Capybara GitHub Action), will cancel the run and trigger the
workflow again when the energy is "least dirty" within specified time constraints. This is called time-shifting - shift
your computation to the times when the grid uses green energy.
It is a simple approach to make your workflows greener without any additional cost.

---

## Composition[](#composition)

When we mention Capybara (not Capybara Action) we have in mind the tool as a whole. Capybara Action is what the user
sees but underneath it calls Capybara backend.

![Capybara flow diagram](docs/diagrams/Capybara%20flow.png)

Capybara tool consists of:

1. **Capybara action** (current repository):
   The action in itself is quite simple: it calls our Capybara backend with the input and cancels the workflow run if it
   wasn't triggered by Capybara backend.
2. **Capybara backend** [CapybaraOrg/capybara-backend](https://github.com/CapybaraOrg/capybara-backend): it is a backend
   service which is handling the requests from the Capybara Action. You will have to host your own instance of this
   backend to use the action.

---

## Usage and Setup[](#usage_and_setup)

Your repository needs to have workflow(s) defined in `.github/workflows` directory (GitHub Actions requirement).

1. To add the action to your repository workflow you need to
   add [`workflow_dispatch`](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch)
   event
   since Capybara backend needs it to trigger the repository workflow.
   It must contain an input parameter `isCapybaraDispatch` which defaults to `false` (this parameter is used internally
   only):

```yaml
on:
  workflow_dispatch:
    inputs:
      isCapybaraDispatch:
        description: System only property (ignore)
        required: true
        default: "false"
```

2. Generate
   a [fine-grained personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token#creating-a-fine-grained-personal-access-token)
   for your repository. In the `Permissions / Repository permissions` section, for `Actions` and `Contents`
   select `Read and write` access.

   Then you will have to register your repository with Capybara backend. To do that, you will have to send an HTTP POST
   request to the Capybara backend's `/v1/accounts` REST endpoint with the GitHub token you just created (future
   development includes supporting better authentication process).

   Example `cURL` request:

```shell
curl \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  https://{CAPYBARA_URL}/v1/accounts \
  -d '{"token": "github_pat_YOUR-TOKEN-HERE"}}'
```

you will receive a response similar to this:

```json
{
  "clientId": "0880cde5-bd95-4145-bd94-3ddbf8d22bfc"
}
```

After that, you will have to add following secrets to your repository where the Capybara GitHub Action is used:

- `CAPYBARA_CLIENT_ID`

  Copy the value of the `clientId` property from the response above.

- `CAPYBARA_URL`

  This will be the Capybara URL of the [CapybaraOrg/capybara-backend](https://github.com/CapybaraOrg/capybara-backend).

For information about secrets and how to add
them [see the docs](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository)

3. Capybara Action should be added as a **first step in the workflow**. This is because, if the run is not triggered by
   Capybara i.e., it's not triggered at the most carbon efficient time, run should be cancelled at the very start.
   If the workflow run **is** triggered by Capybara backend, then the action step will be
   ignored ([see the diagram](./docs/diagrams/Capybara%20flow.png)).

For more information, please see [capybara-demo](https://github.com/CapybaraOrg/capybara-demo) example repository
and [Capybara backend](https://github.com/CapybaraOrg/capybara-backend) repository.

### Inputs

User needs to provide input that is related both to the information about the schedule and the repository itself.
Schedule input is needed for the request to Carbon Aware WebApi. Repository information is needed for triggering the
workflow
with [workflow-dispatch event](https://docs.github.com/en/rest/actions/workflows#create-a-workflow-dispatch-event)

| Parameters              | Required | Type                          | Example values                           | Definition                                                                                                |
| ----------------------- | -------- | ----------------------------- | ---------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `isCapybaraDispatch`    | True     | System only property (ignore) | `${{inputs.isCapybaraDispatch}}`         | System property required by Capybara backend                                                              |
| `capybaraUrl`           | True     | URL of the backend            | `https://{capybara-backend}`             | URL for Capybara backend (https://github.com/CapybaraOrg/capybara-backend). Use it with your own instance |
| `clientId`              | True     | Authentication parameter      | `"0880cde5-bd95-4145-bd94-3ddbf8d22bfc"` | Client id returned by Capybara backend. Please, see the [Usage and Setup](#usage_and_setup)               |
| `workflowId`            | True     | Repository parameter          | `test.yml`                               | The ID of the workflow. You can also pass the workflow file name as a string                              |
| `repoName`              | True     | Repository parameter          | `capybara-demo`                          | The name of the repository. The name is not case sensitive                                                |
| `ref`                   | True     | Repository parameter          | `main`                                   | The git reference for the workflow. The reference can be a branch or tag name                             |
| `owner`                 | True     | Repository parameter          | `JaneDoe`                                | The account owner of the repository. The name is not case sensitive                                       |
| `durationInMinutes`     | False    | Schedule parameter            | `20`                                     | An approximate duration of the workflow run[1]. If not provided, defaults to 5 min                        |
| `maximumDelayInSeconds` | True     | Schedule parameter            | `28800`                                  | How long after the schedule start of the run the pipeline can be triggered[2]                             |
| `location`              | True     | Schedule parameter            | `uksouth`                                | The location of the workflow runner that maps to Azure Cloud Regions[3]                                   |

[1] If the provided duration differs significantly from the actual workflow run duration, the
resulting `bestTimeToStart` output might not be accurate.

[2] Think of it as a window size within which we want the workflow to run. For example, if the run is scheduled with a
cron job to always run at 10PM at night, what is the span in which it's still okay to run the workflow?
Is it 10PM-3AM? 10PM-5AM? Or maybe even 10PM-8AM? In the last case `maximumDelayInSeconds` is equal to `28800`
since `windowSize` is `8` hours (`8` hours in seconds is `28800`).
Mind that the run will finish a bit later - the equation of the latest possible end of the workflow run is:

```math
Latest possible end of the run = bestTimeToStart + maximumDelayInSeconds + durationInMinutes
```

[3] User provides the location where the computation is run, i.e., the location of the runner. This can be derived from
the IP address (if using the self-hosted runner). Otherwise, when using GitHub-hosted runner, deriving the location is a
bit more complex and needs more research.

### Outputs

| Parameters        | Type | Example values        | Definition                                                 |
| ----------------- | ---- | --------------------- | ---------------------------------------------------------- |
| `bestTimeToStart` | Date | `03/11/2022 16:24:16` | Best time to start the workflow run calculated by Capybara |

### Example usage

See the [exemplary workflow in the demo repository](https://github.com/CapybaraOrg/capybara-demo/blob/main/.github/workflows/example-workflow.yml)

---

## Development[](#development)

### Setup[](#setup)

The project is using:

- [Prettier](https://prettier.io/)
- [nvm](https://github.com/nvm-sh/nvm)

```shell
nvm install
npm ci
```

### Publish a new version[](#publish-new-version)

```shell
npm run format
npm run build
```

commit the changes

---

## Future improvements[](#future-improvements)

### Reporting (Capybara dashboard)

We would like extend Capybara with a reporting functionality in a form of a website or even a periodic email containing
a dashboard.

This dashboard would capture the reduced amount of carbon emissions by using Capybara as well as statistics about
particular workflows (e.g., which workflows consume the most carbon).

### Location input not required

Location parameter is automatically derived so that users don't need to provide it. It would be especially useful
if the workflow runs on a GitHub-hosted runner.

### Repository input not required

Repository input could be dynamically read from the client repository. Needs research if it could be done with GitHub context,
e.g., `github.ref_name`.

### Support providing custom input through the workflow dispatch

Right now, we only allow `isCapybaraDispatch` as an input.

### Support multiple jobs in the workflow

As mentioned in the Usage and Setup section, the action will work best when the entire workflow is inside one job. That
is because when we have one job, we only have one runner and therefore one location where the workflow is run.
In the future, we could try to support multiple jobs. With the current implementation, the user can still have multiple
jobs, but the accuracy of the calculated time will be much worse - the time when carbon emissions are lowest at one
location is most probably not the same as in the other location.
See section Gotchas/All steps within on job.

### Authenticate with GitHub via GitHub Apps

Right now, we use fine-grained repository tokens that are sent over to Capybara backend. Ideally, we would
delegate the authentication to GitHub via GitHub Apps.

## Gotchas[](#gotchas) 🤯

### Types of workflow

Since Capybara action is cancelling your current workflow run and running it at later time, it is not suitable for CI/CD
workflows because most CI/CD workflows should be executed immediately.

### All steps within on job

For best results, put your workflow in one job but with multiple steps. This way, we are sure that everything is run in
the same location.
On GitHub, all steps within a job are run on the same runner which ensure they run on the same machine and following
from that - the same location.
If we have multiple jobs, we might calculate the best time for one but not for the others.
