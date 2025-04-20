<script lang='ts'>
  import AutoComplete from 'simple-svelte-autocomplete';
  import { getContext, onDestroy } from 'svelte';

  export let field;
  export let label;
  export let validation;
  export let placeholder;
  export let dataSourceType;
  export let searchEvent;
  export let apiUrl;
  export let queryParam;
  export let staticOptions;
  export let dataProvider;
  export let loading;
  export let results;
  export let labelFieldName;
  export let valueFieldName;

  let resultsPromise;
  let loadingResolver;

  const { styleable } = getContext('sdk');
  const component = getContext('component');
  const formContext = getContext('form');
  const formStepContext = getContext('form-step');
  const fieldGroupContext = getContext('field-group');

  let fieldState;
  let fieldApi;
  let text = '';
  let selectedItem;
  let initialItemsPromise;

  const formApi = formContext?.formApi;
  const labelPos = fieldGroupContext?.labelPosition || 'above';
  $: formStep = formStepContext ? $formStepContext || 1 : 1;
  $: labelClass = labelPos === 'above' ? '' : `spectrum-FieldLabel--${labelPos}`;
  $: delay = dataSourceType === 'static' ? 0 : 100;

  $: dataContext = {
    text,
    value: selectedItem?.[valueFieldName]
  };

  // Init du champ de formulaire
  $: if (formApi && field) {
    const formField = formApi.registerField(field, 'text', '', false, validation, formStep);
    unsubscribe = formField?.subscribe(async (fieldValue) => {
      fieldState = fieldValue?.fieldState;
      fieldApi = fieldValue?.fieldApi;

      const value = fieldState?.value;
      if (value && !initialItemsPromise) {
        console.log('[🔄] Loading initial items for', value);
        placeholder = 'Loading...';
        initialItemsPromise = await getItems(value);
        const item = (await initialItemsPromise).find(i => i[valueFieldName] === value);

        selectedItem = item ?? {
          [valueFieldName]: value,
          [labelFieldName]: value
        };

        placeholder = '';
      }
    });
  }

  // Gestion du loading de query (Budibase query mode)
  $: if (dataSourceType === 'query' || (dataSourceType === 'budibase' && searchEvent)) {
    const parsedLoading = loading?.toLowerCase?.() === 'true' || loading === '1';
    if (parsedLoading && !loadingResolver) {
      console.log('[🔁] Loading results from query...');
      resultsPromise = new Promise((resolve) => {
        loadingResolver = resolve;
      });
    } else if (!parsedLoading && loadingResolver) {
      try {
        console.log('[✅] Resolving query results:', results);
        const parsedResults =
          dataSourceType === 'query' ? JSON.parse(results || '[]') : dataProvider?.rows;
        console.log('[📦] Parsed results:', parsedResults);
        loadingResolver(parsedResults || []);
        loadingResolver = null;
      } catch (err) {
        console.error('[❌] Failed to parse results:', err);
        loadingResolver([]);
        loadingResolver = null;
      }
    }
  }

  // Force static config
  $: if (dataSourceType === 'static') {
    labelFieldName = 'label';
    valueFieldName = 'value';
  }

  onDestroy(() => {
    fieldApi?.deregister();
    unsubscribe?.();
  });

  async function getItems(keyword: string) {
    console.log('[🔍] getItems called with keyword:', keyword);

    switch (dataSourceType) {
      case 'budibase':
        if (!searchEvent) {
          console.log('[📡] Budibase - returning static rows');
          return dataProvider?.rows || [];
        }

      case 'query':
        if (searchEvent) {
          console.log('[📨] Firing search event for:', keyword);
          searchEvent({ keyword });
        }
        await new Promise(resolve => setTimeout(resolve, 100));
        return await resultsPromise;

      case 'static':
        console.log('[📋] Using static options');
        return staticOptions;

      case 'api':
        try {
          const url = new URL(apiUrl);
          url.searchParams.set(queryParam, keyword);
          console.log('[🌍] Fetching from API:', url.toString());
          const response = await fetch(url);
          const json = await response.json();
          console.log('[📥] API response:', json);
          return json;
        } catch (e) {
          console.error('[❌] Error fetching from API:', e);
          return [];
        }

      default:
        console.warn('[❓] Unknown dataSourceType:', dataSourceType);
        return [];
    }
  }

  function changeHandler(e) {
    console.log('[🧠] selectedItem:', selectedItem);
    console.log('[🧠] changeHandler event:', e);
    if (selectedItem) {
      fieldApi?.setValue(selectedItem[valueFieldName]);
    }
  }
</script>

<div class='spectrum-Form-item' class:above={labelPos === "above"} use:styleable={$component.styles}>
  {#if !formContext}
    <div class='placeholder'>Form components need to be wrapped in a form</div>
  {:else}
    <label
      class:hidden={!label}
      for={fieldState?.fieldId}
      class={`spectrum-FieldLabel spectrum-FieldLabel--sizeM spectrum-Form-itemLabel ${labelClass}`}
    >
      {label || ' '}
    </label>
    <div class='spectrum-Form-itemField'>
      <AutoComplete
        inputId={fieldState?.fieldId}
        className="autocomplete-width"
        bind:selectedItem
        bind:text
        {placeholder}
        {labelFieldName}
        {valueFieldName}
        {delay}
        cleanUserText={false}
        searchFunction={getItems}
        onChange={changeHandler}
      />
    </div>
    {#if fieldState?.error}
      <div class='error'>{fieldState.error}</div>
    {/if}
  {/if}
</div>

<style>
  .placeholder {
    color: var(--spectrum-global-color-gray-600);
  }

  label {
    white-space: nowrap;
  }

  label.hidden {
    padding: 0;
  }

  .spectrum-FieldLabel--right,
  .spectrum-FieldLabel--left {
    padding-right: var(--spectrum-global-dimension-size-200);
  }

  .spectrum-Form-item.above {
    display: flex;
    flex-direction: column;
  }

  .spectrum-Form-itemField {
    position: relative;
    width: 100%;
  }

  :global(.autocomplete-width) {
    width: 100%;
  }
</style>
