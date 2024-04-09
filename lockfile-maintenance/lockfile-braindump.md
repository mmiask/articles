## Warning: all of the mentioned below is based on true events. No developers were harmed during implementation and/or for the purposes of this article

### **Reason**:

We faced an issue where we weren't tracking the current version of a dependency. In `package.json`, it had a caret `^`, which caused to instal the latest version for new users, but at the same time we had an old one locally (and were testing against it).

1. There was a dependency (A) that is used by another dependency(B): we bumped the free-floating one. Then decided it's worth bumping dependencyB instead of doing this. It turned out that there are some failures happening in there, but not in our project (Elements). This made me wonder if test our library correctly.
Problem: should we test only the library or the dependencies we control as well?

### **Solution 1:**

We can use Dependabot to bump denpendencies weekly and run our CI workflow against that.

### **Solution 2: (actually this is solution 3)**
We can write our own script that runs once a week and create PRs?

### **Problem 1:**

Dependabot doesn't allow to have duplicate configs, e.g. you can't have weekly updates for `dependencies` and daily for `devDependencies`

---

### **Topics**

- Yarn & npm
- yarn.lock
- package.json
- releases in npm
- Dependabot and similar
- multiple packages handled by one repo?
- semantic versioner

---
### Step 1: learn more about `package.json` and the lockfile
1. What is `Yarn`? Differencies between `Yarn` and `npm` (link to some articles)
2. `Yarn 2`?
3. What is the process of installing dependencies? Is it based on package.json, yarn.lock, both?

### Step 2: release
1. Dependency versions in repo vs package.
> **Answer**: `yarn.lock` is not a part of a npm package, so users install the fresh set of versions
2. Can users affect installed npm package?
3. Do we want to keep the dependency versions in package.json exact and release after bumps, or make them up-to-date semantically and test periodically?

### Solution 1 - Dependabot
- what is Dependabot?
- what it is good for?
- what is it missing? why we decided not to use it?

### Solution 2 - Renovate
- Overview of functionalities
- Mentioning Docs
- App vs self-hosted?
- Possible future usage?

### Solution 3 - CircleCI workflow
- Writing the workflow
- using gh api
- testing using Elements pipeline

### Aftermath - updating the CircleCI job