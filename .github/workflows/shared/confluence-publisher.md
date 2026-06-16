---
import-schema:
  confluence-base-url:
    type: string
    required: true
    description: "Base URL of your Atlassian instance (e.g., https://mycompany.atlassian.net)"
  confluence-space-key:
    type: string
    required: true
    description: "Confluence space key where documentation pages are published (e.g., DOCS, ENG)"
  parent-page-id:
    type: string
    required: false
    default: ""
    description: "Optional Confluence parent page ID to nest documentation under a specific page"

safe-outputs:
  jobs:
    publish-to-confluence:
      description: "Create or update a documentation page in Confluence via REST API. The agent must provide a title and content in Confluence storage format (XHTML)."
      runs-on: ubuntu-latest
      inputs:
        title:
          description: "Page title for the Confluence documentation page"
          type: string
          required: true
        content:
          description: "Page body in Confluence storage format (XHTML). Use valid Confluence macros and HTML tags."
          type: string
          required: true
      permissions:
        contents: read
      env:
        CONFLUENCE_BASE_URL: ${{ github.aw.import-inputs.confluence-base-url }}
        CONFLUENCE_SPACE_KEY: ${{ github.aw.import-inputs.confluence-space-key }}
        CONFLUENCE_PARENT_PAGE_ID: ${{ github.aw.import-inputs.parent-page-id }}
        CONFLUENCE_USER_EMAIL: ${{ secrets.CONFLUENCE_USER_EMAIL }}
        CONFLUENCE_API_TOKEN: ${{ secrets.CONFLUENCE_API_TOKEN }}
      steps:
        - name: Publish documentation to Confluence
          uses: actions/github-script@v8
          with:
            script: |
              const fs = require('fs');
              const staged = process.env.GH_AW_SAFE_OUTPUTS_STAGED === 'true';
              const outputFile = process.env.GH_AW_AGENT_OUTPUT;
              const data = JSON.parse(fs.readFileSync(outputFile, 'utf8'));
              const items = (data.items || []).filter(i => i.type === 'publish_to_confluence');

              if (items.length === 0) {
                core.warning('No publish_to_confluence items found in agent output.');
                return;
              }

              const baseUrl = (process.env.CONFLUENCE_BASE_URL || '').replace(/\/+$/, '');
              const spaceKey = process.env.CONFLUENCE_SPACE_KEY;
              const parentPageId = process.env.CONFLUENCE_PARENT_PAGE_ID || '';
              const email = process.env.CONFLUENCE_USER_EMAIL;
              const token = process.env.CONFLUENCE_API_TOKEN;

              if (!baseUrl || !email || !token || !spaceKey) {
                core.setFailed('Missing required Confluence configuration. Verify CONFLUENCE_BASE_URL, CONFLUENCE_SPACE_KEY, CONFLUENCE_USER_EMAIL, and CONFLUENCE_API_TOKEN.');
                return;
              }

              const auth = Buffer.from(`${email}:${token}`).toString('base64');
              const headers = {
                'Authorization': `Basic ${auth}`,
                'Content-Type': 'application/json',
                'Accept': 'application/json'
              };

              for (const item of items) {
                const { title, content } = item;
                if (!title || !content) {
                  core.warning('Skipping item with missing title or content.');
                  continue;
                }

                if (staged) {
                  core.info(`[STAGED] Would publish page: "${title}" to space ${spaceKey}`);
                  core.summary.addRaw(`## Staged: Confluence Publish\n**Title:** ${title}\n**Space:** ${spaceKey}\n\n\`\`\`html\n${content.substring(0, 1000)}\n\`\`\``);
                  await core.summary.write();
                  continue;
                }

                // Search for existing page by title in the target space
                const searchParams = new URLSearchParams({
                  title: title,
                  spaceKey: spaceKey,
                  expand: 'version'
                });
                const searchUrl = `${baseUrl}/wiki/rest/api/content?${searchParams}`;
                const searchRes = await fetch(searchUrl, { headers });

                if (!searchRes.ok) {
                  const errText = await searchRes.text();
                  core.setFailed(`Confluence search failed (${searchRes.status}): ${errText}`);
                  return;
                }

                const searchData = await searchRes.json();

                if (searchData.results && searchData.results.length > 0) {
                  // Update existing page — increment version number
                  const page = searchData.results[0];
                  const newVersion = page.version.number + 1;
                  const putUrl = `${baseUrl}/wiki/rest/api/content/${page.id}`;
                  const putBody = {
                    id: page.id,
                    type: 'page',
                    title: title,
                    space: { key: spaceKey },
                    body: {
                      storage: {
                        value: content,
                        representation: 'storage'
                      }
                    },
                    version: { number: newVersion }
                  };

                  const putRes = await fetch(putUrl, {
                    method: 'PUT',
                    headers,
                    body: JSON.stringify(putBody)
                  });

                  if (!putRes.ok) {
                    const errText = await putRes.text();
                    core.setFailed(`Failed to update page "${title}" (${putRes.status}): ${errText}`);
                    return;
                  }

                  core.info(`Updated Confluence page: "${title}" (ID: ${page.id}, version: ${newVersion})`);
                } else {
                  // Create new page
                  const postUrl = `${baseUrl}/wiki/rest/api/content`;
                  const postBody = {
                    type: 'page',
                    title: title,
                    space: { key: spaceKey },
                    body: {
                      storage: {
                        value: content,
                        representation: 'storage'
                      }
                    }
                  };

                  if (parentPageId) {
                    postBody.ancestors = [{ id: parentPageId }];
                  }

                  const postRes = await fetch(postUrl, {
                    method: 'POST',
                    headers,
                    body: JSON.stringify(postBody)
                  });

                  if (!postRes.ok) {
                    const errText = await postRes.text();
                    core.setFailed(`Failed to create page "${title}" (${postRes.status}): ${errText}`);
                    return;
                  }

                  const created = await postRes.json();
                  core.info(`Created Confluence page: "${title}" (ID: ${created.id})`);
                }
              }
---

<!-- Confluence Publisher — Shared Safe-Output Component -->
<!-- Provides a deterministic post-agent job that publishes documentation to Confluence via REST API. -->
<!-- Install in consumer repos: gh aw add https://github.com/{org}/agentic-workflows/blob/main/.github/workflows/shared/confluence-publisher.md -->
