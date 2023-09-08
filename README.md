# GitHub Actions overwrite cache example repo

GitHub Actions does not support overwrite cache with same key.

- [Feature request: option to update cache · Issue #342 · actions/cache](https://github.com/actions/cache/issues/342)

As workaround, you can use `actions/cache/restore` and [gh-actions-cache](https://github.com/actions/gh-actions-cache), and `actions/cache/save`. 

This workflow implements overwrite cache using restore + delete + save.

```yaml
name: Update Cache
on:
  workflow_dispatch:
permissions:
  contents: read
  actions: write # require to delete cache
jobs:
  calendar:
    runs-on: ubuntu-latest
    env:
      # overwrite cache key
      cache-key: your-cache-key
    steps:
      # This job implements overwrite cache using restore + delete + save
      - name: Checkout
        uses: actions/checkout@v3 # gh command require repository
      - name: Restore Cache
        id: cache-restore
        uses: actions/cache/restore@v3
        with:
          path: ./cache
          key: ${{ env.cache-key }}
      # Main Task
      - name: Main Task
        run: |
          # generate current time to ./cache/time
          mkdir -p ./cache
          previous_date=$(cat ./cache/time || echo "No previous date")
          current_date=$(date +%s)
          echo "Previous: $previous_date"
          echo "Current: $current_date"
          # Save current time to ./cache/time
          echo "$current_date" > ./cache/time
        env:
          CACHE_DIR: ./cache
      # overwrite cache key: delete previous and save current
      - name: Delete Previous Cache
        if: ${{ steps.cache-restore.outputs.cache-hit }}
        continue-on-error: true
        run: |
          gh extension install actions/gh-actions-cache
          if ${{ steps.cache-restore.outputs.cache-hit == 'true' }}; then
            gh actions-cache delete "${{ env.cache-key }}" --confirm
          fi
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: Save Cache
        uses: actions/cache/save@v3
        with:
          path: ./cache
          key: ${{ env.cache-key }}
```
