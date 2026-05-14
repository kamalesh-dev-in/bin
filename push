#!/bin/bash

if [ -z "$1" ]; then
  echo "Usage: push \"commit message\""
  exit 1
fi

if ! git rev-parse --is-inside-work-tree > /dev/null 2>&1; then
  echo "Error: not inside a git repository"
  exit 1
fi

git add . || exit 1
git commit -m "$1" || exit 1
git push
