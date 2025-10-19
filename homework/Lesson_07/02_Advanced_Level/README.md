1. Take refference from gitops
2. Modify to use OCIRepository instead of GitRepository
3. Add missing ResourceSetInputProviders
4. Update kbot-hr.yaml to use correct namespaces and ResourceSet resource
5. Override chart and image versions using ResourceSetInputProviders
6. Validate the configuration
7. Commit and push the changes to the repository

# Related repositories
- https://github.com/vkamlov/flux-gitops-dev/tree/main
- https://github.com/vkamlov/kbot-src/tree/main
