<script lang="ts">
  import AutoComplete from 'simple-svelte-autocomplete';
  import { getContext, onDestroy, onMount } from 'svelte';

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

  let unsubscribe = null;

  console.log("✅ Composant monté");

  const { styleable } = getContext('sdk');
  const component = getContext('component');
  const formContext = getContext('form');
  const formStepContext = getContext('form-step');
  const fieldGroupContext = getContext('field-group');

  let fieldState;
  let fieldApi;
  let unsubscribe;
  let text = "";
  let selectedItem;
  let resultsPromise;
  let loadingResolver;
  let initialItemsPromise;

  const formApi = formContext?.formApi;
  const labelPos = fieldGroupContext?.labelPosition || 'above';
  $: formStep = formStepContext ? $formStepContext || 1 : 1;
  $: labelClass = labelPos === 'above' ? '' : `spectrum-FieldLabel--${labelPos}`;

  $: console.log("📊 field:", field);
  $: console.log("📊 dataSourceType:", dataSourceType);
  $: console.log("📊 formContext:", formContext);

  $: dataContext = {
    text,
    value: selectedItem?.[valueFieldName]
  };

  $: delay = dataSourceType === 'static' ? 0 : 100;

  $: if (formApi && field) {
    const formField = formApi.registerField(
      field,
      'text',
      '',
      false,
      validation,
      formStep
    );

    unsubscribe = formField?.subscribe(async (fieldValue) => {
      fieldState = fieldValue?.fieldState;
      fieldApi = fieldValue?.fieldApi;
      const value = fieldState?.value;

      if (value && !initialItemsPromise) {
        console.log('📦 Loading initial items');
        placeholder = 'Loading';
        initialItemsPromise = await getItems(value);
        const item = (await initialItemsPromise).find((item) => item[valueFieldName] === value);

        if (item) {
          selectedItem = item;
        } else {
          selectedItem = {
            [valueFieldName]: value,
            [labelFieldName]: value
          };
        }
        placeholder = '';
      }
    });
  }

  $: if (dataSourceType === 'query' || (dataSourceType === 'budibase' && searchEvent)) {
  const parsedLoading = loading?.toLowerCase() === 'true' || loading === '1';

  if (parsedLoading && !loadingResolver) {
    console.log('⏳ Chargement des résultats (promise init)');
    resultsPromise = new Promise((resolve) => {
      loadingResolver = resolve;
    });
  } else if (!parsedLoading && loadingResolver) {
    console.log('✅ Résultats dispo, traitement...');

    try {
      let parsedResults = [];

      if (dataSourceType === 'query') {
        parsedResults = Array.isArray(JSON.parse(results)) 
          ? JSON.parse(results) 
          : [];
      } else {
        parsedResults = Array.isArray(dataProvider?.rows)
          ? dataProvider.rows
          : [];
      }

      console.log('📦 Résultats parsés :', parsedResults);
      loadingResolver(parsedResults);
    } catch (err) {
      console.error('❌ Erreur lors du parsing de results', err);
      loadingResolver([]);
    }

    loadingResolver = null;
  }
}


  $: if (dataSourceType === 'static') {
    labelFieldName = 'label';
    valueFieldName = 'value';
  }

  onDestroy(() => {
    fieldApi?.deregister();
    if (typeof unsubscribe === "function") {
        unsubscribe();
    }
});

  async function getItems(keyword: string) {
    console.log('🔎 getItems called with keyword:', keyword);
    switch (dataSourceType) {
      case 'budibase':
        if (!searchEvent) return dataProvider?.rows || [];
      case 'query':
        if (searchEvent) searchEvent({ keyword });
        await new Promise((resolve) => setTimeout(resolve, 100));
        return await resultsPromise;
      case 'static':
        return staticOptions;
      case 'api':
        const url = new URL(apiUrl);
        url.searchParams.set(queryParam, keyword);
        const response = await fetch(url);
        const result = await response.json();
        console.log('🌐 Fetched API result:', result);
        return result;
      default:
        console.warn('⚠️ dataSourceType not recognized');
        return [];
    }
  }

  function changeHandler(e) {
    console.log('🎯 changeHandler called, selectedItem:', selectedItem);
    if (selectedItem) {
      fieldApi?.setValue(selectedItem[valueFieldName]);
    }
  }
</script>

<div class='spectrum-Form-item' class:above={labelPos === "above"} use:styleable={$component.styles}>
  {#if !formContext}
    <div class='placeholder'>🚨 Form context not found</div>
  {:else}
    <label class:hidden={!label} for={fieldState?.fieldId} class={`spectrum-FieldLabel spectrum-FieldLabel--sizeM spectrum-Form-itemLabel ${labelClass}`}>
      {label || ' '}
    </label>
    <div class='spectrum-Form-itemField'>
      <AutoComplete
        inputId={fieldState?.fieldId}
        className="autocomplete-width"
        searchFunction={getItems}
        bind:selectedItem
        bind:text
        {placeholder}
        {labelFieldName}
        {valueFieldName}
        onChange={changeHandler}
        {delay}
        cleanUserText={false}
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
