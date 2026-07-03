# Funnel execute examples

## create scaffolded funnel

```js
async () => {
  const randomUUID = () => crypto.randomUUID();
  const projectId = randomUUID();

  const funnel = await codemode.funnels_create({
    name: 'Sales funnel',
    description: 'Created from the sales template',
    projectId,
    domainId: null
  });

  const sequence = await codemode.funnels_sequence_create({
    funnelId: funnel.id,
    template: 'sales',
    position: { x: 0, y: 0 }
  });

  const validation = await codemode.funnels_validate({ funnelId: funnel.id });
  const structure = await codemode.funnels_structure_get({ funnelId: funnel.id });

  return {
    funnel,
    nodes: sequence.nodes || [],
    structure,
    validation
  };
};
```

## build custom graph

```js
async () => {
  const randomUUID = () => crypto.randomUUID();
  const funnelId = randomUUID();

  const optin = await codemode.funnels_node_create({
    funnelId,
    type: 'page',
    title: 'Opt-in',
    slug: 'optin',
    pageType: 'capture',
    position: { x: 0, y: 0 },
    triggers: [{ type: 'form_submit', label: 'Form submitted' }]
  });

  const sales = await codemode.funnels_node_create({
    funnelId,
    type: 'page',
    title: 'Sales page',
    slug: 'sales',
    pageType: 'sales',
    position: { x: 360, y: 0 },
    triggers: [
      { type: 'button_click', label: 'Buy button clicked' },
      { type: 'purchase_approved', label: 'Purchase approved' }
    ]
  });

  const thanks = await codemode.funnels_node_create({
    funnelId,
    type: 'page',
    title: 'Thank you',
    slug: 'thank-you',
    pageType: 'thank_you',
    position: { x: 720, y: 0 },
    triggers: []
  });

  const connected = await codemode.funnels_triggers_connect({
    funnelId,
    connections: [
      {
        nodeId: optin.node.id,
        triggerId: optin.node.triggers[0].id,
        target: sales.node.id
      },
      {
        nodeId: sales.node.id,
        triggerId: sales.node.triggers.find((trigger) => trigger.type === 'purchase_approved').id,
        target: thanks.node.id
      }
    ]
  });

  return {
    connectedTriggers: connected.triggers,
    structure: await codemode.funnels_structure_get({ funnelId }),
    validation: await codemode.funnels_validate({ funnelId })
  };
};
```

## link pages validate publish

```js
async () => {
  const randomUUID = () => crypto.randomUUID();
  const funnelId = randomUUID();

  const pageLinks = [
    { nodeId: randomUUID(), pageId: randomUUID() },
    { nodeId: randomUUID(), pageId: randomUUID() }
  ];

  const linkedPages = [];
  for (const link of pageLinks) {
    linkedPages.push(
      await codemode.funnels_node_connect_page({
        funnelId,
        nodeId: link.nodeId,
        pageId: link.pageId
      })
    );
  }

  const validation = await codemode.funnels_validate({ funnelId });
  const structure = await codemode.funnels_structure_get({ funnelId });
  const publishableNodeIds = (structure.nodes || []).filter((node) => node.type !== 'notes').map((node) => node.id);

  const published = await codemode.funnels_publish({
    funnelId,
    nodeIds: publishableNodeIds
  });

  return {
    linkedPages,
    validation,
    publishableNodeIds,
    published
  };
};
```
