<%*
// === PaperMod Theme Optimized Template ===
// This template creates posts with full PaperMod theme support

function translit(str) {
    const map = {
        'а':'a','б':'b','в':'v','г':'g','д':'d','е':'e','ё':'yo','ж':'zh',
        'з':'z','и':'i','й':'y','к':'k','л':'l','м':'m','н':'n','о':'o',
        'п':'p','р':'r','с':'s','т':'t','у':'u','ф':'f','х':'h','ц':'c',
        'ч':'ch','ш':'sh','щ':'shch','ъ':'','ы':'y','ь':'','э':'e','ю':'yu','я':'ya'
    };
    return str.toLowerCase()
        .trim()
        .replace(/\s+/g, '-')
        .split('')
        .map(c => map[c] ?? c)
        .join('')
        .replace(/[^a-z0-9\-_.]/g, '')
        .replace(/-+/g, '-')
        .replace(/^-+|-+$/g, '');
}

function getAllFolders() {
    const out = [];
    const root = app.vault.getRoot();
    function walk(folder) {
        out.push(folder.path);
        if (!folder.children) return;
        folder.children.forEach(c => {
            if (Array.isArray(c.children)) walk(c);
        });
    }
    walk(root);
    return out;
}

try {
    const folders = getAllFolders();
    if (!folders || folders.length === 0) {
        new Notice("Failed to get folder list from vault.");
        return;
    }

    // Select base folder
    const basePath = await tp.system.suggester(folders, folders, false, "Select folder for post:");
    if (!basePath) {
        new Notice("Folder selection cancelled.");
        return;
    }

    const today = tp.date.now("YYYY-MM-DD");
    const postTitle = await tp.system.prompt("Enter post title:");
    if (!postTitle) {
        new Notice("Title not entered - cancelled.");
        return;
    }

    const slug = translit(postTitle) || "untitled";
    const folderName = `${today}-${slug}`;
    const finalPath = `${basePath}/${folderName}`;

    // Create folder for post
    await app.vault.createFolder(finalPath).catch(()=>{});

    // Create attachments folder
    const attachmentsPath = `${finalPath}/attachments`;
    await app.vault.createFolder(attachmentsPath).catch(()=>{});
    await app.vault.create(`${attachmentsPath}/.gitkeep`, '').catch(()=>{});

    // Category from path
    const categoryName = basePath.split("/").pop();

    // === PaperMod-specific options ===

    // Ask about cover image
    const wantCover = await tp.system.suggester(
        ["Yes, add cover image", "No, no cover image"],
        [true, false],
        false,
        "Add cover image?"
    );

    let coverSection = '';
    if (wantCover) {
        const coverFile = await tp.system.prompt("Cover image filename (e.g., cover.jpg):", "attachments/cover.png");
        const coverHidden = await tp.system.suggester(
            ["Show on page", "Hide on page (for OpenGraph only)"],
            [false, true],
            false,
            "Show cover image on page?"
        );

        coverSection = `cover:
  image: ${coverFile}
  alt: "${postTitle}"
  caption: ""
  relative: false
  hidden: ${coverHidden}`;
    }

    // Ask about table of contents (TOC)
    const showToc = await tp.system.suggester(
        ["Yes, show table of contents", "No, no table of contents"],
        [true, false],
        false,
        "Show table of contents?"
    );

    const tocOpen = showToc ? await tp.system.suggester(
        ["Collapsed", "Expanded"],
        [false, true],
        false,
        "Default table of contents state:"
    ) : false;

    // Tags
    const tagsInput = await tp.system.prompt("Tags (comma separated):", "ai");
    const tags = tagsInput ? tagsInput.split(',').map(t => t.trim()).filter(t => t) : ["ai"];

    // Description
    const description = await tp.system.prompt("Brief post description:", "Post description.");

    // Generate frontmatter
    const frontmatter = `---
draft: true
title: ${JSON.stringify(postTitle)}
date: ${today}
lastmod:
author: Aleksei Aksenov

# URL settings
slug: ${slug}
url: /${categoryName}/${slug}
aliases: []

# Taxonomies
categories:
  - ${categoryName}
tags:
${tags.map(t => `  - ${t}`).join('\n')}

# PaperMod features
ShowToc: ${showToc}
TocOpen: ${tocOpen}
weight: 1
hidemeta: false
comments: false
description: ${JSON.stringify(description)}
disableShare: false
hideSummary: false
searchHidden: false

${coverSection ? coverSection : '# cover: # Uncomment to add cover image\n#   image: attachments/cover.png\n#   alt: ""\n#   caption: ""\n#   relative: false\n#   hidden: false'}
---

## Introduction

${postTitle}

## Main Content

Write content here.

## Conclusion

Summary and conclusions.
`;

    // Create index.md
    const targetPath = `${finalPath}/index.md`;
    const existing = app.vault.getAbstractFileByPath(targetPath);
    if (existing) {
        new Notice(`⚠️ Already exists: ${targetPath}`);
        await app.workspace.getLeaf().openFile(existing);
    } else {
        const newFile = await app.vault.create(targetPath, frontmatter);
        new Notice(`✅ Created: ${targetPath}`);
        await app.workspace.getLeaf().openFile(newFile);
    }

} catch (err) {
    new Notice("Script error: " + (err?.message ?? String(err)));
    console.error(err);
}
%>
