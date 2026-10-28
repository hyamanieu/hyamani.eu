# AGENTS.md

## Context

This file describes how agents should help develop this website. The website is built with Hugo using the Blowfish theme. A Blowfish skill is accessible.

## Debugging and verifying

Always work within the CWD, i.e. the repository root folder where this file is located. Generated files are located under ./public

During programming, `hugo server -D` is usually running and ./public is automatically updated. You can verify it with `ps aux | grep "hugo server -D"`.

If it's not running, ask your human to run it. If they say no, simply run `hugo --minify`.