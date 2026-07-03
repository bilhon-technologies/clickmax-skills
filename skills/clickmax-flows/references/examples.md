# Flow execute examples

## create -> trigger -> action

```js
async () => {
  const randomUUID = () => crypto.randomUUID();
  const projectId = randomUUID();
  const tagId = randomUUID();

  const flow = await codemode.flows_create({
    name: 'Boas-vindas novos leads',
    projectId,
    category: 'marketing'
  });

  const trigger = await codemode.flows_step_create({
    flowId: flow.id,
    type: 'trigger',
    input: { type: 'event' },
    triggerStart: [
      {
        type: 'start',
        eventName: 'crm.tag.lead.apply.v1',
        constraints: [{ field: 'tagId', op: 'EQ', valueType: 'UUID', valueUuid: tagId }]
      }
    ]
  });

  const tagStep = await codemode.flows_step_create({
    flowId: flow.id,
    type: 'action',
    action: 'addTag',
    input: { tagId }
  });

  await codemode.flows_step_connect({
    flowId: flow.id,
    stepId: trigger.step.id,
    target: tagStep.step.id
  });

  return await codemode.flows_validate({ flowId: flow.id });
};
```

## trigger -> delay -> send_message

```js
async () => {
  const randomUUID = () => crypto.randomUUID();
  const flowId = randomUUID();
  const tagId = randomUUID();

  const trigger = await codemode.flows_step_create({
    flowId,
    type: 'trigger',
    input: { type: 'event' }
  });

  await codemode.flows_step_triggers_set({
    flowId,
    stepId: trigger.step.id,
    triggerStart: [
      {
        eventName: 'crm.tag.lead.apply.v1',
        constraints: [{ field: 'tagId', op: 'EQ', valueType: 'UUID', valueUuid: tagId }]
      }
    ]
  });

  const delay = await codemode.flows_step_create({
    flowId,
    type: 'delay',
    input: { when: 24 }
  });

  // Sender id (telegramBotId) is optional at creation — omit it and the user picks the bot in
  // the builder before activating. message.type is required ('text' for telegram).
  const message = await codemode.flows_step_create({
    flowId,
    type: 'send_message',
    input: {
      type: 'telegram',
      message: { type: 'text', content: 'Obrigado! Ficamos felizes em ter voce por aqui.' }
    }
  });

  await codemode.flows_step_connect({ flowId, stepId: trigger.step.id, target: delay.step.id });
  await codemode.flows_step_connect({ flowId, stepId: delay.step.id, target: message.step.id });

  return {
    structure: await codemode.flows_structure_get({ flowId }),
    validation: await codemode.flows_validate({ flowId })
  };
};
```

## conditional -> true/false branch

```js
async () => {
  const randomUUID = () => crypto.randomUUID();
  const flowId = randomUUID();
  const checkTagId = randomUUID();
  const vipListId = randomUUID();
  const applyTagId = randomUUID();

  const cond = await codemode.flows_step_create({
    flowId,
    type: 'conditional',
    input: {
      mode: 'AND',
      statements: [{ type: 'tagApplied', tagId: checkTagId, operator: 'is' }]
    }
  });

  const ifTagged = await codemode.flows_step_create({
    flowId,
    type: 'action',
    action: 'addContactToList',
    input: { listId: vipListId }
  });

  const ifNotTagged = await codemode.flows_step_create({
    flowId,
    type: 'action',
    action: 'addTag',
    input: { tagId: applyTagId }
  });

  await codemode.flows_step_connect({ flowId, stepId: cond.step.id, handle: 'true', target: ifTagged.step.id });
  await codemode.flows_step_connect({ flowId, stepId: cond.step.id, handle: 'false', target: ifNotTagged.step.id });

  return await codemode.flows_structure_get({ flowId });
};
```

## inspect -> validate

```js
async () => {
  const randomUUID = () => crypto.randomUUID();
  const flowId = randomUUID();

  const structure = await codemode.flows_structure_get({ flowId });
  const validation = await codemode.flows_validate({ flowId });

  return {
    startStepId: structure.startStepId,
    steps: (structure.steps || []).map((step) => ({
      id: step.id,
      type: step.type,
      action: step.action || null,
      isEntry: step.isEntry
    })),
    edges: structure.edges || [],
    validation
  };
};
```
