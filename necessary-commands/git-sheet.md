**Create new branch from main:**

```bash
git switch main
git pull origin main
git switch -c branch_name
```

**before switch you could also run** 
```bash
go mod tidy
slqc generate
```

**Update existing branch with main:**

```bash
git switch main
git pull origin main
git switch branch_name
git merge main
```

**before switch you could also run** 
```bash
go mod tidy
slqc generate
```

