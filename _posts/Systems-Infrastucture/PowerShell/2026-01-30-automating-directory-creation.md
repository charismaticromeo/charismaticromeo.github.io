---
title: "Automating Directory Creation Across Multiple Paths in PowerShell"
date: 2026-01-30
categories: 
  - Systems-Infrastructure
  - PowerShell
tags:
  - notes
  - automation
  - filesystem
---

Have you ever found yourself needing to create the same subdirectory structure across multiple directories? If so, you already know it’s one of the most repetitive and mind-numbing tasks you can do.

Brace expansion on Linux makes it trivial. You just need to:

```bash
mkdir -p base_dir{1,2}/sub{1..3}
```

That single command gives you:

```
├── base_dir1
│   ├── sub1
│   ├── sub2
│   └── sub3
└── base_dir2
    ├── sub1
    ├── sub2
    └── sub3
```

Unfortunately, PowerShell doesn’t have a direct equivalent to Bash’s brace expansion. But that doesn’t mean we’re stuck doing things manually. With a small, reusable script, we can achieve the same result and make it flexible enough to reuse in the future.

In this post, we’ll walk through **how to solve this on Windows using PowerShell**.

## The Use Case

For this example, we’ll assume we’re setting up a tech blog repository with the following high-level categories:
* **Databases & Data** (PostgreSQL, MySQL, Redis, BigQuery)
* **Systems & Infrastructure** (PowerShell, Docker, Kubernetes, Linux)
* **Low-level & Embedded** (Assembly, C, Rust, Microcontrollers)

Each topic should contain two standard subfolders:
* Notes
* Demos

> This kind of predictable structure future-proofs your repository and makes collaboration and maintenance much easier.
{: .prompt-info }

---

## The PowerShell Script

```powershell
# Creates Category -> Subcategory -> (Notes, Demos) structure for a tech blog

$rootPath = '.'

$structure = @{
    'Databases and Data' = @('PostgreSQL', 'MySQL', 'Redis', 'BigQuery')
    'Systems and Infrastructure' = @('PowerShell', 'Docker', 'Kubernetes', 'Linux')
    'Low-level and Embedded' = @('Assembly', 'C', 'Rust', 'Microcontrollers')
}

$subFolders = @('Notes', 'Demos')

foreach ($category in $structure.Keys) {
    $catPath = Join-Path $rootPath $category

    # Create category folder if missing
    if (-not (Test-Path $catPath)) {
        New-Item -Path $catPath -ItemType Directory | Out-Null
        Write-Host "Created category: $catPath" -ForegroundColor Cyan
    }

    foreach ($subcat in $structure[$category]) {
        $subcatPath = Join-Path $catPath $subcat

        # Create subcategory folder
        if (-not (Test-Path $subcatPath)) {
            New-Item -Path $subcatPath -ItemType Directory | Out-Null
            Write-Host "  └── Created: $subcat" -ForegroundColor Green
        }

        # Create standard subfolders inside each topic
        foreach ($sub in $subFolders) {
            $finalPath = Join-Path $subcatPath $sub

            if (-not (Test-Path $finalPath)) {
                New-Item -Path $finalPath -ItemType Directory | Out-Null
                Write-Host "      ├── Created: $sub" -ForegroundColor DarkGreen
            }
        }
    }
}

Write-Host "`nBlog content structure ready! Happy writing." -ForegroundColor Yellow
```
{: file="Create-TechBlogStructure.ps1" }

## How to Run

1. Open PowerShell and navigate to the directory where you want the structure created.
2. Save the script as `Create-TechBlogStructure.ps1`.
3. Temporarily allow script execution:
   ```powershell
   Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process
   ```
4. Run the script:
   ```powershell
   .\Create-TechBlogStructure.ps1
   ```
5. That’s it. Our entire directory structure is created automatically.

## Example Output

```
Created category: .\Low-level and Embedded
  └── Created: Assembly
      ├── Created: Notes
      ├── Created: Demos
  └── Created: C
      ├── Created: Notes
      ├── Created: Demos
  ...
Created category: .\Systems and Infrastructure
  └── Created: PowerShell
      ├── Created: Notes
      ├── Created: Demos
  └── Created: Docker
      ├── Created: Notes
      ├── Created: Demos
  ...
```

> **Tip:**
> Want a different structure or additional categories? Just edit the `$structure` hashtable.
> The script is idempotent—you can safely re-run it anytime without overwriting existing folders.
{: .prompt-tip }

---


