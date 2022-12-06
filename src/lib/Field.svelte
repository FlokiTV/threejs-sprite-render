<script lang="ts">
  export let title = "";
  export let value = "";
</script>


<div class="form-control w-full max-w-xs">
  <label class="label">
    <span class="label-text">{title}</span>
    <span class="label-text-alt">{value}</span>
  </label>
  <slot />
</div>