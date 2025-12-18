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
        new Notice("Не удалось получить список папок в хранилище.");
        return;
    }

    // Выбор базовой папки
    const basePath = await tp.system.suggester(folders, folders, false, "Выберите папку для поста:");
    if (!basePath) {
        new Notice("Выбор папки отменён.");
        return;
    }

    const today = tp.date.now("YYYY-MM-DD");
    const rusName = await tp.system.prompt("Введите название поста (на русском):");
    if (!rusName) {
        new Notice("Название не введено — отменено.");
        return;
    }

    const slug = translit(rusName) || "untitled";
    const folderName = `${today}-${slug}`;
    const finalPath = `${basePath}/${folderName}`;

    // Создаём папку для поста
    await app.vault.createFolder(finalPath).catch(()=>{});

    // Создаём папку attachments
    const attachmentsPath = `${finalPath}/attachments`;
    await app.vault.createFolder(attachmentsPath).catch(()=>{});
    await app.vault.create(`${attachmentsPath}/.gitkeep`, '').catch(()=>{});

    // Категория из пути
    const categoryName = basePath.split("/").pop();

    // === PaperMod-specific options ===

    // Спрашиваем про обложку
    const wantCover = await tp.system.suggester(
        ["Да, добавить обложку", "Нет, без обложки"],
        [true, false],
        false,
        "Добавить обложку (cover image)?"
    );

    let coverSection = '';
    if (wantCover) {
        const coverFile = await tp.system.prompt("Имя файла обложки (например, cover.jpg):", "cover.jpg");
        const coverHidden = await tp.system.suggester(
            ["Показывать на странице", "Скрыть на странице (только для OpenGraph)"],
            [false, true],
            false,
            "Показывать обложку на странице?"
        );

        coverSection = `cover:
  image: ${coverFile}
  alt: "${rusName}"
  caption: ""
  relative: false
  hidden: ${coverHidden}`;
    }

    // Спрашиваем про оглавление (TOC)
    const showToc = await tp.system.suggester(
        ["Да, показывать оглавление", "Нет, без оглавления"],
        [true, false],
        false,
        "Показывать оглавление (Table of Contents)?"
    );

    const tocOpen = showToc ? await tp.system.suggester(
        ["Свёрнутое", "Развёрнутое"],
        [false, true],
        false,
        "Оглавление по умолчанию:"
    ) : false;

    // Теги
    const tagsInput = await tp.system.prompt("Теги (через запятую):", "ai");
    const tags = tagsInput ? tagsInput.split(',').map(t => t.trim()).filter(t => t) : ["ai"];

    // Описание
    const description = await tp.system.prompt("Краткое описание поста:", "Описание поста.");

    // Формируем фронтматтер
    const frontmatter = `---
draft: true
title: ${JSON.stringify(rusName)}
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

${coverSection ? coverSection : '# cover: # Uncomment to add cover image\n#   image: cover.jpg\n#   alt: ""\n#   caption: ""\n#   relative: false\n#   hidden: false'}
---

## Введение

${rusName}

## Основное содержание

Напишите содержание здесь.

## Заключение

Итоги и выводы.
`;

    // Создаём index.md
    const targetPath = `${finalPath}/index.md`;
    const existing = app.vault.getAbstractFileByPath(targetPath);
    if (existing) {
        new Notice(`⚠️ Уже существует: ${targetPath}`);
        await app.workspace.getLeaf().openFile(existing);
    } else {
        const newFile = await app.vault.create(targetPath, frontmatter);
        new Notice(`✅ Создано: ${targetPath}`);
        await app.workspace.getLeaf().openFile(newFile);
    }

} catch (err) {
    new Notice("Ошибка скрипта: " + (err?.message ?? String(err)));
    console.error(err);
}
%>
