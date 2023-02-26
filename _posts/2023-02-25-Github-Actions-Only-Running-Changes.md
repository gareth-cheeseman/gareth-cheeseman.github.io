---
layout: post
title:  "Complex conditional running of github actions jobs"
date:   2023-02-25 16:40:01
categories: post
---
{% newthought 'Only run jobs you need' %} in github actions when you have specific changes in your repository.<!--more--> 

Do you have a repository with several projects that have different builds or other tasks in your Continuous Integration (CI)? Is your CI running all the jobs when irrelevant code is changed?

It will save you time, and money, to only run those jobs when the relevant code has changed.

### Detecting changes in files

The great *Paths Changes Filter* action{% sidenote 'paths-filter' 'See [dorny/paths-filter](https://github.com/dorny/paths-filter#paths-changes-filter)' %} solves the most difficult part of this – working out which files have changed. This action has a comprehensive README so go there to get full details; but you may end up with something like this in your workflow file:

```yaml
jobs:
  has_file_changes:
    runs-on: ubuntu-20.04
    outputs:
      {% raw %}packages: ${{ steps.dinner_filter.outputs.changes }}{% endraw %}
    steps:
      - uses: dorny/paths-filter@v2.11.1
        id: dinner_filter
        with:
          filters: |
            main_dish:
              - 'main_dish/**'
            veggie_side:
              - 'veggie_side/**'
            dessert:
              - 'fruit_salad/**'
              - 'choco_pud/**'
```
Note the `output:` this is so we can use the result in other jobs. The `packages` output is a JSON array. Any of the filters that have file changes will be present in the array e.g. `[main_dish, fruit_salad]`.

### Using the changes output in other jobs
You can then use the output in other jobs, such as a build. For instance if each of the dinner project needs to have a different build image you could use it in a matrix strategy like this:

{% marginnote 'danny' 'Thanks to my friend [Danny Teixeira](https://www.linkedin.com/in/danny-teixeira-5a388835/) for showing me this pattern.' %}
```yaml 
  {% raw %}build:
    runs-on: ubuntu-20.04
    needs: has_file_changes
    if: | 
      ${{
         needs.has_file_changes.outputs.packages != '[]' &&
         needs.has_file_changes.outputs.packages != ''
      }}
    strategy:
      matrix:
        package: ${{ fromJSON(needs.has_file_changes.outputs.packages) }}
    steps:
      - name: Set Variables
        run: |
          PROJECT_PATH=$(echo ${{ matrix.package }})
          echo "PROJECT_PATH=$PROJECT_PATH" >> $GITHUB_ENV
      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v3
        with:
          push: true
          tags: your_image_store/${{ env.PROJECT_PATH }}:latest
          file: ${{ env.PROJECT_PATH }}/Dockerfile{% endraw %}
```
Note the `if` condition requires that the output isn't empty. If the output is empty the job won't run. If the job does run, the JSON array is parsed with the github actions `fromJSON` function{% sidenote 'fromJSON', 'See [fromJSON](https://docs.github.com/en/actions/learn-github-actions/expressions#fromjson)' %}.

### More complicated output consumption
But what if things are more complicated? Perhaps one build actually builds two projects, but both projects need testing separately if there's a change? We can do two filter steps and combine the outputs where needed.

```yaml
{% raw %}jobs:
  has_file_changes:
    runs-on: ubuntu-20.04
    outputs:
      build: ${{ steps.build_filter.outputs.changes }}
      test: ${{ steps.combined_test_filter.outputs.changes }}
    steps:
      - uses: dorny/paths-filter@v2.11.1
        id: build_filter
        with:
          filters: |
            dessert:
              - 'fruit_salad/**'
              - 'choco_pud/**'
      - uses: dorny/paths-filter@v2.11.1
        id: test_filter
        with:
          filters: |
            choco_pud:
              - 'choco_pud/**'
      - name: Combined test
        id: combined_test_filter
        env:
          BUILD: ${{ steps.build_filter.outputs.changes }}
          TEST: ${{ steps.test_filter.outputs.changes }}
        run: |
          SAFE_BUILD=${BUILD:=[]}
          SAFE_TEST=${TEST:=[]}
          JSON="{ \"build\": $SAFE_BUILD, \"test\": $SAFE_TEST }"
          echo "changes=$(jq ".build+.test" <<< "$JSON" -rc)" >> $GITHUB_OUTPUT{% endraw %}
```
We can then use the outputs in our follow up jobs. For the build: `needs.has_file_changes.outputs.build` and for the test: `needs.has_file_changes.outputs.test`.

We use *jq*{% sidenote 'jq', 'See [jq](https://stedolan.github.io/jq/)' %}, a JSON processor, to add the arrays together (*jq* is already installed in github runners). But because of that we need to do few preparatory steps. *jq* expects a JSON input. I couldn't work out the syntax to process two JSON arrays that weren't within an object, so I constructed a JSON string. And use a *here-string* `<<<`{% sidenote 'here-string', 'See section 3.6.7 [Bash Manual](https://www.gnu.org/software/bash/manual/html_node/Redirections.html)' %} to pass it to *jq*.  A gotcha here, is that *jq* expects an array and the filter output can be an empty string. So add some 'safe' variables by assiging an empty array as a default value{% sidenote 'default-value', 'See section 3.5.3 [Bash Manual](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html)' %}. Finally we echo that to the `GITHUB_OUTPUT` so it's available in subsequent jobs.