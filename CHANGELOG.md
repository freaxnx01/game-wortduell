# Changelog

All notable changes to this project are documented here, following
[Keep a Changelog](https://keepachangelog.com) and
[Semantic Versioning](https://semver.org).

## [0.2.0] - 2026-07-27

### Added

- Hide Hint button outside vs-computer mode
- Make played words in the move log open a Google search
- Add in-game Rules overlay (DE/EN)

### Documentation

- Add design spec for rules screen, hint gating, word-search links
- Add implementation plan for rules screen, hint gating, word links
- Document rules, word validation, and dictionary source

### Fixed

- Tolerate legacy log entries without a words array on Resume
- Remove non-official 2-letter noise from the bundled base list

## [Unreleased]

### Added
- Hover over the version number to see when the game was last updated, in your own time zone, and how long ago

### Fixed
- The version number in the game navigation was empty: the page re-rendered the navigation shortly after loading and dropped it. It is shown again and links to the changelog. A second, invisible version badge left over from before the navigation existed is gone

## [0.3.0] - 2026-09-29

### Added
- Fullscreen toggle (⛶) in the game navigation

## [0.1.0] - 2026-07-19

### Added

- Initial versioned release of game-wortduell.
- In-game version badge sourced from `version.js`.
