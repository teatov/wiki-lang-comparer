<script lang="ts">
  type SiteMatrixItem = {
    code: string;
    name: string;
    site: {
      url: string;
      dbname: string;
      code: string;
      sitename: string;
      closed?: string;
    };
    dir: string;
    localname: string;
  };

  type SiteMatrix = {
    sitematrix: Record<string, SiteMatrixItem>;
  };

  type Language = {
    code: string;
    name: string;
    key: string;
    title: string;
    localname?: string;
  };

  type Article = {
    name: string;
    languages: Language[];
  };

  let sourceArticleURL = $state<string>(
    'https://en.wikipedia.org/wiki/Main_Page'
  );
  let targetArticleURL = $state<string>(
    'https://en.wikipedia.org/wiki/United_States'
  );

  let isFetching = $state<boolean>(false);
  let infoMessage = $state<string | null>(null);
  let errorMessage = $state<string | null>(null);
  let langDiff = $state<Language[] | null>(null);
  let listCopied = $state<boolean>(false);

  async function doTheThing() {
    isFetching = true;
    errorMessage = null;
    infoMessage = null;
    langDiff = null;
    listCopied = false;

    try {
      const info = await fetchInfo();

      infoMessage = 'Finding the difference in languages...';
      langDiff = info.source.languages
        .filter((sourceLang) =>
          info.target.languages.every(
            (targetLang) => sourceLang.code !== targetLang.code
          )
        )
        .map((lang) => ({
          ...lang,
          localname: Object.values(info.siteMatrix.sitematrix).find(
            (item) => item.code === lang.code
          )?.localname,
        }));

      infoMessage = `<strong>${info.source.name}</strong> has <strong>${info.source.languages.length}</strong> languages, and <strong>${info.target.name}</strong> has <strong>${info.target.languages.length}</strong>.<br /><strong>${info.source.name}</strong> has <strong>${langDiff.length}</strong> languages that <strong>${info.target.name}</strong> doesn't have.`;
    } catch (err) {
      console.error(err);
      isFetching = false;
      errorMessage = String(err);
      langDiff = null;
      return;
    }

    isFetching = false;
  }

  async function fetchInfo() {
    infoMessage = 'Parsing the source article URL...';
    const sourceArticle = parseArticleURL(new URL(sourceArticleURL));

    infoMessage = 'Parsing the target article URL...';
    const targetArticle = parseArticleURL(new URL(targetArticleURL));

    infoMessage = 'Fetching Wikipedia SiteMatrix...';
    const siteMatrixResponse = await fetch(
      'https://commons.wikimedia.org/w/api.php?action=sitematrix&smtype=language&format=json&origin=*'
    );
    const siteMatrix = (await siteMatrixResponse.json()) as SiteMatrix;

    infoMessage = 'Fetching languages for the source article...';
    const sourceArticleLangs = await getArticleLanguages(
      sourceArticle.code,
      sourceArticle.name
    );

    infoMessage = 'Fetching languages for the target article...';
    const targetArticleLangs = await getArticleLanguages(
      targetArticle.code,
      targetArticle.name
    );

    return {
      source: {
        name: sourceArticle.name,
        languages: sourceArticleLangs,
      } as Article,
      target: {
        name: targetArticle.name,
        languages: targetArticleLangs,
      } as Article,
      siteMatrix,
    };
  }

  function parseArticleURL(articleURL: URL) {
    const domains = articleURL.hostname.split('.');
    if (domains.length < 3) {
      throw new Error(
        `Hostname ${articleURL.hostname} does not have 3 levels.`
      );
    }
    let code = domains[0];

    const pathname = articleURL.pathname.split('/');
    if (pathname.length < 1) {
      throw new Error(
        `Pathname ${articleURL.pathname} has less than 1 segment.`
      );
    }
    let name = pathname[pathname.length - 1];
    return { code, name };
  }

  async function getArticleLanguages(
    code: string,
    name: string
  ): Promise<Language[]> {
    const response = await fetch(
      `https://api.wikimedia.org/core/v1/wikipedia/${code}/page/${name}/links/language`
    );
    const languages = (await response.json()) as Language[];
    return languages;
  }

  function swapArticles() {
    const temp = sourceArticleURL;
    sourceArticleURL = targetArticleURL;
    targetArticleURL = temp;
  }

  function copyListToClipboard() {
    if (!langDiff) return;
    const listString = langDiff
      .map((lang) => `${lang.localname} (${lang.name})`)
      .join('\n');
    navigator.clipboard.writeText(listString);
    listCopied = true;
  }
</script>

<main class="flex h-full flex-col items-center justify-evenly">
  <div class="max-w-xl p-4">
    <p class="mb-4">
      Find the languages that one Wikipedia article has, but the other doesn't.
    </p>
    <label for="source-article-url" class="block mb-2">
      Source article URL
      <input
        type="text"
        class="w-full border p-2 disabled:cursor-not-allowed"
        bind:value={sourceArticleURL}
        disabled={isFetching}
        name="source-article-url"
        id="source-article-url"
      />
    </label>
    <div class="text-center">
      <button
        class="px-4 py-1 bg-foreground text-background"
        onclick={swapArticles}
        disabled={isFetching}>Swap</button
      >
    </div>
    <label for="target-article-url" class="block mb-2">
      Target article URL
      <input
        type="text"
        class="w-full border p-2 disabled:cursor-not-allowed"
        bind:value={targetArticleURL}
        disabled={isFetching}
        name="target-article-url"
        id="target-article-url"
      />
    </label>
    <button
      class="p-2 bg-foreground text-background block w-full disabled:cursor-not-allowed mb-2"
      onclick={doTheThing}
      disabled={isFetching}>Do the thing</button
    >
    {#if infoMessage}<p>{@html infoMessage}</p>{/if}
    {#if errorMessage}<p class="text-danger">{@html errorMessage}</p>{/if}
    {#if langDiff}
      <table class="mt-4 mb-2 w-full">
        <thead>
          <tr class="m-0 p-0">
            <th class="border px-2 py-1 font-bold">Language</th>
            <th class="border px-2 py-1 font-bold">Native name</th>
            <th class="border px-2 py-1 font-bold">Source article</th>
          </tr>
        </thead>
        <tbody class="border-b">
          {#each langDiff as lang}
            <tr class="m-0 p-0 odd:bg-secondary/15">
              <td class="border-x px-2">{lang.localname}</td>
              <td class="border-x px-2">{lang.name}</td>
              <td class="border-x px-2"
                ><a
                  href={`https://${lang.code}.wikipedia.org/wiki/${lang.key}`}
                  class="font-semibold underline">{lang.title}</a
                ></td
              >
            </tr>
          {/each}
        </tbody>
      </table>
      <button
        class="p-2 bg-foreground text-background"
        onclick={copyListToClipboard}
        >Copy list to clipboard {#if listCopied}
          (Copied!)
        {/if}</button
      >
    {/if}
    <p class="mt-4">
      Made by <a href="https://teatov.xyz" class="font-semibold underline"
        >Teatov</a
      >. Source code available on
      <a
        href="https://github.com/teatov/wiki-lang-comparer"
        class="font-semibold underline">GitHub</a
      >.
    </p>
  </div>
</main>
