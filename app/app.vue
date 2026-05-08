<script setup lang="ts">
useHead({
  meta: [
    { name: 'viewport', content: 'width=device-width, initial-scale=1' }
  ],
  link: [
    { rel: 'icon', href: '/favicon.ico' }
  ],
  htmlAttrs: {
    lang: 'en'
  }
})

const title = 'Income Tax Calculator (Massachusetts)'
const description = 'Calculate net income and state and federal taxes for a single full-year Massachusetts resident.'

useSeoMeta({
  title,
  description,
  ogTitle: title,
  ogDescription: description,
  twitterCard: 'summary_large_image'
})
</script>

<template>
  <UApp>
    <div class="flex min-h-screen flex-col">
      <UHeader class="shrink-0">
        <template #left>
          <NuxtLink to="/">
            <span class="flex items-center gap-2 text-sm font-semibold text-highlighted">
              <UIcon
                name="i-lucide-calculator"
                class="size-5 text-primary"
              />
              Income Tax Calculator (Massachusetts)
            </span>
          </NuxtLink>
        </template>

        <template #right>
          <UColorModeButton />
        </template>
      </UHeader>

      <main class="min-h-0 flex-1">
        <NuxtPage />
      </main>

      <footer class="shrink-0 border-t border-default py-3">
        <UContainer>
          <p class="text-center text-sm text-muted">
            Copyright &copy; {{ new Date().getFullYear() }} Upamanyu Pathare. Licensed under the GNU Affero General Public License v3.0.
          </p>
        </UContainer>
      </footer>
    </div>
  </UApp>
</template>
