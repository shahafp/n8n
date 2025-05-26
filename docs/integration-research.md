# n8n Integrations and Workflow Import

This document summarizes how integrations are represented in the n8n codebase and how workflows can be imported via JSON.

## Integrations as Nodes

- Each integration is implemented as an n8n **node**. All core nodes live under `packages/nodes-base/nodes`. Example directory listing:

```
ActionNetwork
ActiveCampaign
AcuityScheduling
Adalo
Affinity
AgileCrm
AiTransform
Airtable
Airtop
Amqp
```

- Each integration folder contains a TypeScript implementation and a `.node.json` file describing metadata. The start of the Airtable node JSON shows links to integration documentation:

```json
{       "node": "n8n-nodes-base.airtable",
        "nodeVersion": "1.0",
        "codexVersion": "1.0",
        "categories": ["Data & Storage"],
        "resources": {
                "credentialDocumentation": [
                        {
                                "url": "https://docs.n8n.io/integrations/builtin/credentials/airtable/"
                        }
                ],
                "primaryDocumentation": [
```

- The Airtable node TypeScript class registers available versions:

```ts
import type { INodeTypeBaseDescription, IVersionedNodeType } from 'n8n-workflow';
import { VersionedNodeType } from 'n8n-workflow';

import { AirtableV1 } from './v1/AirtableV1.node';
import { AirtableV2 } from './v2/AirtableV2.node';

export class Airtable extends VersionedNodeType {
        constructor() {
                const baseDescription: INodeTypeBaseDescription = {
                        displayName: 'Airtable',
                        name: 'airtable',
                        icon: 'file:airtable.svg',
                        group: ['input'],
                        description: 'Read, update, write and delete data from Airtable',
                        defaultVersion: 2.1,
                };

                const nodeVersions: IVersionedNodeType['nodeVersions'] = {
                        1: new AirtableV1(baseDescription),
                        2: new AirtableV2(baseDescription),
                        2.1: new AirtableV2(baseDescription),
                };

                super(nodeVersions, baseDescription);
        }
}
```

## Workflow Structure

- Workflows are JSON objects containing an array of `nodes` and a `connections` object. When constructing a workflow, the `Workflow` class loads node types and fills in default parameters:

```ts
// Also directly add the default values of the node type.
let nodeType: INodeType | undefined;
for (const node of parameters.nodes) {
        this.nodes[node.name] = node;

        nodeType = this.nodeTypes.getByNameAndVersion(node.type, node.typeVersion);

        if (nodeType === undefined) {
                // Go on to next node when its type is not known.
                // For now do not error because that causes problems with
                // expression resolution also then when the unknown node
                // does not get used.
                continue;
                // throw new ApplicationError(`Node with unknown node type`, {
                //      tags: { nodeType: node.type },
                //      extra: { node },
                // });
        }

        // Add default values
        const nodeParameters = NodeHelpers.getNodeParameters(
                nodeType.description.properties,
                node.parameters,
                true,
                false,
                node,
                nodeType.description,
        );
        node.parameters = nodeParameters !== null ? nodeParameters : {};
```

- The example workflow JSON `cypress/fixtures/Onboarding_workflow.json` shows nodes and their connections:

```json
      ]
    },
    "connections": {
      "HubSpot Trigger": {
        "main": [
          [
            {
              "node": "IF",
              "type": "main",
              "index": 0
            }
          ]
        ]
      },
      "IF": {
        "main": [
          [
            {
              "node": "Is Gmail, Don't Add to Sheet",
              "type": "main",
              "index": 0
            }
          ],
          [
            {
              "node": "Google Sheets",
              "type": "main",
              "index": 0
            }
          ]
        ]
      }
    },
    "active": false,
```

## Importing Workflows from JSON

### CLI

- n8n provides a command `n8n import:workflow` to import workflows from JSON files. Supported options include `--input` for the file or directory, `--separate` to read multiple files, and optional `--userId` or `--projectId` for ownership.

```ts
export class ImportWorkflowsCommand extends BaseCommand {
        static description = 'Import workflows';

        static examples = [
                '$ n8n import:workflow --input=file.json',
                '$ n8n import:workflow --separate --input=backups/latest/',
                '$ n8n import:workflow --input=file.json --userId=1d64c3d2-85fe-4a83-a649-e446b07b3aae',
                '$ n8n import:workflow --input=file.json --projectId=Ox8O54VQrmBrb4qL',
                '$ n8n import:workflow --separate --input=backups/latest/ --userId=1d64c3d2-85fe-4a83-a649-e446b07b3aae',
        ];
```

### HTTP API

- The REST controller accepts a URL and validates that it returns a proper workflow JSON before importing:

```ts
async getFromUrl(
                _req: AuthenticatedRequest,
                _res: express.Response,
                @Query query: ImportWorkflowFromUrlDto,
) {
        let workflowData: IWorkflowResponse | undefined;
        try {
                const { data } = await axios.get<IWorkflowResponse>(query.url);
                workflowData = data;
        } catch (error) {
                throw new BadRequestError('The URL does not point to valid JSON file!');
        }

        // Do a very basic check if it is really a n8n-workflow-json
        if (
                workflowData?.nodes === undefined ||
                !Array.isArray(workflowData.nodes) ||
                workflowData.connections === undefined ||
                typeof workflowData.connections !== 'object' ||
                Array.isArray(workflowData.connections)
        ) {
                throw new BadRequestError(
                        'The data in the file does not seem to be a n8n workflow JSON file!',
                );
        }

        return workflowData;
}
```

### Frontend

- Cypress tests demonstrate importing workflows in the UI from a URL or from a file:

```ts
before(() => {
        cy.fixture('Onboarding_workflow.json').then((data) => {
                cy.intercept('GET', '/rest/workflows/from-url*', {
                        body: { data },
                }).as('downloadWorkflowFromURL');
        });
});
...
workflowPage.getters.workflowMenuItemImportFromURLItem().click();
workflowPage.getters.inputURLImportWorkflowFromURL().should('be.visible');
workflowPage.getters
        .inputURLImportWorkflowFromURL()
        .type('https://fakepage.com/workflow.json');
workflowPage.getters.confirmActionImportWorkflowFromURL().click();
```

```ts
workflowPage.getters.workflowMenuItemImportFromFile().click();
workflowPage.getters
        .workflowImportInput()
        .selectFile('fixtures/Test_workflow-actions_paste-data.json', { force: true });
cy.waitForLoad(false);
workflowPage.actions.zoomToFit();
workflowPage.getters.canvasNodes().should('have.length', 5);
workflowPage.getters.nodeConnections().should('have.length', 5);
```

These tests open the import dialog, select a JSON file or provide a URL, and check that nodes show on the canvas.

## Custom Integrations

- The contributing guide states that PRs adding new nodes are auto-closed unless requested by the n8n team but users can create their own nodes following the documentation:

```
- **New Nodes:**
  - PRs that introduce new nodes will be **auto-closed** unless they are explicitly requested by the n8n team and aligned with an agreed project scope. However, you can still explore [building your own nodes](https://docs.n8n.io/integrations/creating-nodes/) , as n8n offers the flexibility to create your own custom nodes.
```

```
## Create custom nodes

Learn about [building nodes](https://docs.n8n.io/integrations/creating-nodes/) to create custom nodes for n8n. You can create community nodes and make them available using [npm](https://www.npmjs.com/).
```

## Summary

- Integrations in n8n are individual node packages located under `packages/nodes-base/nodes`.
- Node JSON files reference documentation pages, and node classes register versions and features.
- A workflow JSON lists `nodes` and their `connections`, processed by the `Workflow` class.
- Workflows can be imported via CLI (`n8n import:workflow`) or through the REST API/UI, as shown in the controller and Cypress tests.
- Developers can build custom integrations by creating new nodes following the docs linked in the contributing guide.
