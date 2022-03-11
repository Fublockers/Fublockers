- 👋 Hi, I’m @Fublockers
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...

<!---
Fublockers/Fublockers is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
name: Receive PR

# read-only repo token
# no access to secrets
on:
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:        
      - uses: actions/checkout@v2

      # imitation of a build process
      - name: Build
        run: /bin/bash ./build.sh

      - name: Save PR number
        run: |
          mkdir -p ./pr
          echo ${{ github.event.number }} > ./pr/NR
      - uses: actions/upload-artifact@v2
        with:
          name: pr
          path: pr/
CommentPR.yml

name: Comment on the pull request

# read-write repo token
# access to secrets
on:
  workflow_run:
    workflows: ["Receive PR"]
    types:
      - completed

jobs:
  upload:
    runs-on: ubuntu-latest
    if: >
      ${{ github.event.workflow_run.event == 'pull_request' &&
      github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: 'Download artifact'
        uses: actions/github-script@v3.1.0
        with:
          script: |
            var artifacts = await github.actions.listWorkflowRunArtifacts({
               owner: context.repo.owner,
               repo: context.repo.repo,
               run_id: ${{github.event.workflow_run.id }},
            });
            var matchArtifact = artifacts.data.artifacts.filter((artifact) => {
              return artifact.name == "pr"
            })[0];
            var download = await github.actions.downloadArtifact({
               owner: context.repo.owner,
               repo: context.repo.repo,
               artifact_id: matchArtifact.id,
               archive_format: 'zip',
            });
            var fs = require('fs');
            fs.writeFileSync('${{github.workspace}}/pr.zip', Buffer.from(download.data));
      - run: unzip pr.zip

      - name: 'Comment on PR'
        uses: actions/github-script@v3
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            var fs = require('fs');
            var issue_number = Number(fs.readFileSync('./NR'));
            await github.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: issue_number,
              body: 'Everything is OK. Thank you for the PR!'
            });
