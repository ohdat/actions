# ations

通用 action

## docker-ci

```yaml
- name: build
  uses: ohdat/actions/.github/workflows/docker-ci.yml@master
  secrets:
    DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
```
