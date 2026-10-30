<script setup>
defineProps({ name: String })
const paths = {
  mail: 'M3 5h18v14H3z M3 5l9 8 9-8',
  inbox: 'M4 4h16v16H4z M4 14h5l1 3h4l1-3h5',
  draft: 'M5 3h14v18H5z M8 8h8 M8 12h8 M8 16h5',
  sent: 'M3 3l19 9-19 9 4-9z M7 12h15',
  spam: 'M8 3h8l5 5v8l-5 5H8l-5-5V8z M12 7v6 M12 17h.01',
  trash: 'M3 6h18 M9 6V3h6v3 M5 6l1 15h12l1-15 M10 10v7 M14 10v7',
  star: 'M12 3l2.8 5.7 6.2.9-4.5 4.4 1.1 6.2-5.6-3-5.6 3 1.1-6.2L3 9.6l6.2-.9z',
  search: 'M20 20l-5-5 M17 10a7 7 0 1 1-14 0 7 7 0 0 1 14 0',
  plus: 'M12 5v14 M5 12h14',
  refresh: 'M20 7v5h-5 M4 17v-5h5 M5 8a8 8 0 0 1 13-3l2 3 M4 16l2 3a8 8 0 0 0 13-3',
  clip: 'M8 13l6-6a3 3 0 0 1 4 4l-8 8a5 5 0 0 1-7-7l9-9 M6 15l8-8',
  reply: 'M9 5l-7 7 7 7v-5c7 0 10 2 13 6-1-9-5-12-13-12z',
  forward: 'M15 5l7 7-7 7v-5H2v-4h13z',
  back: 'M15 5l-7 7 7 7',
  next: 'M9 5l7 7-7 7',
  close: 'M6 6l12 12 M6 18L18 6',
  menu: 'M4 6h16 M4 12h16 M4 18h16',
  download: 'M12 3v12 M7 10l5 5 5-5 M4 16v5h16v-5',
  restore: 'M4 11a8 8 0 1 1 1 6 M4 4v7h7 M12 7v5l3 2',
}
</script>

<template>
  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path :d="paths[name] || paths.mail" /></svg>
</template>
