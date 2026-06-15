# Contribution 1: #Tools > File Editor should give more descriptive error messages
**Contribution Number:** 1
**Student:** Eugene Lotsu
**Issue:** [Issue #4333](https://github.com/Yoast/wordpress-seo/issues/4333)
**Status:** Phase IV Complete

---

## Why I Chose This Issue
I chose this issue because it is a beginner-friendly contribution that focuses on improving the user experience. The current error message does not clearly explain whether the problem is caused by insufficient permissions or a filesystem issue. By improving the error handling and messaging, users will receive more accurate feedback, making the application easier to understand and troubleshoot. This issue also gives me an opportunity to learn more about the codebase and contribute to an open-source project in a meaningful way.

---

## Understanding the Issue

### Problem Description
When a user without the `edit_files` capability navigates to the Yoast SEO File Editor, they receive no meaningful feedback. The File Editor link is silently hidden on the Tools page, and if accessed directly, the page shows blank or a generic WordPress error. The issue asks that the plugin clearly communicate whether the problem is a permissions issue or a filesystem issue.

### Expected Behavior
When a user without `edit_files` visits the File Editor, they should see a clear message such as: "You do not have sufficient permissions to edit system files." Similarly, if the file cannot be written due to a filesystem problem, that should also be communicated clearly.

### Current Behavior
- The File Editor link is silently hidden from users without `edit_files`
- If accessed directly, the page shows blank or a generic WordPress "Sorry, you are not allowed to access this page." error
- No distinction is made between a permissions problem and a filesystem problem

### Affected Components
- `admin/pages/tools.php` — controls whether the File Editor link appears and whether the page loads
- `admin/views/tool-file-editor.php` — the File Editor page itself
- `inc/class-wpseo-utils.php` — contains `allow_system_file_edit()` which gates access based on `edit_files` capability

---

## Reproduction Process

### Environment Setup
- **Local dev tool:** LocalWP on macOS
- **WordPress version:** 7.0
- **Yoast SEO version:** 18.0 (installed manually via zip — newer versions removed the File Editor entirely)
- **PHP:** 8.2

**Challenges faced:**
- TasteWP (online sandbox) had the file editor disabled — switched to LocalWP
- The cloned repo (v27.8) had the File Editor removed — had to install v18.0 manually
- `composer install` failed initially because composer was not installed — fixed with `brew install composer`
- `npm install` failed due to peer dependency conflicts — fixed with `npm install --legacy-peer-deps`
- `npm run build` failed due to a lerna config error — not needed since the fix is PHP-only; copied individual files instead
- The file was named `tool-file-editor.php`, not `file-editor.php` as initially assumed

### Steps to Reproduce
1. Install Yoast SEO 18.0 on a local WordPress site (download from https://downloads.wordpress.org/plugin/wordpress-seo.18.0.zip)
2. Create a user with the **SEO Manager** role (added by Yoast)
3. Log in as that user
4. Navigate to **SEO > Tools**
5. Observe: File Editor link is missing with no explanation
6. Navigate directly to `wp-admin/admin.php?page=wpseo_tools&tool=file-editor`
7. Observe: Blank page or generic WordPress error — no permission message

### Reproduction Evidence
- **Branch link:** https://github.com/lotsuegene/wordpress-seo/tree/fix-issue-4333
- **My findings:** The silence is caused by `WPSEO_Utils::allow_system_file_edit()` returning `false` for users without `edit_files`, which causes both the link and the page to be blocked without any user-facing message.

---

## Solution Approach

### Analysis
The root cause is in two places:

1. **`admin/pages/tools.php`** — The File Editor link and page access are both wrapped in `if ( WPSEO_Utils::allow_system_file_edit() === true )`. When this returns `false`, the link disappears silently and the page never loads.

2. **`admin/views/tool-file-editor.php`** — There is no top-level permission check. Permission is only checked inside individual POST handlers, so loading the page as a restricted user produces no output.

3. **`WPSEO_Utils::allow_system_file_edit()`** — Returns `false` for any user without `edit_files`, including SEO Manager, causing silent failures upstream.

### Proposed Solution
- In `tools.php`: Show the File Editor entry to all users, but display a "no permissions" message in the description for users who lack `edit_files`. Also allow the page to load so the permission check inside it can run.
- In `tool-file-editor.php`: Add a top-level permission check at the start of the file that shows a clear error and exits early.

### Implementation Plan

**Understand:**
Users without `edit_files` get no feedback when trying to access the File Editor. The plugin silently hides or blocks it. The fix should show a clear, descriptive message instead.

**Match:**
The existing POST handlers in `tool-file-editor.php` already use `current_user_can( 'edit_files' )` checks — the top-level check follows the same pattern. The tools list in `tools.php` already supports custom `desc` fields per tool.

**Plan:**
1. Modify `admin/pages/tools.php` to show the File Editor entry with a permissions message when `allow_system_file_edit()` returns `false`
2. Modify `admin/pages/tools.php` to allow `file-editor` to load regardless of permissions (so the view-level check can run)
3. Add a top-level permission check in `admin/views/tool-file-editor.php` before any POST handling or rendering

**Implement:** https://github.com/lotsuegene/wordpress-seo/tree/fix-issue-4333

**Review:**
- Follows existing code style (tabs, same echo patterns, same translation functions)
- Does not remove existing POST-level permission checks (defense in depth)
- Uses `esc_html_e()` for output escaping consistent with surrounding code
- Does not break behavior for users who do have `edit_files`

**Evaluate:**
- Log in as SEO Manager → SEO > Tools → File Editor entry now shows with permissions message 
- Click File Editor link → "You do not have sufficient permissions to edit files." message shown 
- Log in as Administrator → File Editor works as normal 

---

## Testing Strategy

### Manual Testing
- [] SEO Manager user sees File Editor listed on Tools page with permissions message
- [] SEO Manager user clicking File Editor link sees clear permission error
- [] Administrator user sees no change in behavior — File Editor works normally
- [] Verified fix is in the correct file (`tool-file-editor.php`, not `file-editor.php`)

### Unit Tests
- [ ] Test that `allow_system_file_edit()` returns `false` for users without `edit_files`
- [ ] Test that tools page shows File Editor entry for all users
- [ ] Test that File Editor page returns permission error for users without `edit_files`

### Integration Tests
- [ ] SEO Manager role cannot edit files but sees descriptive error
- [ ] Administrator role can access and use File Editor normally

---

## Implementation Notes

### Code Changes

**File 1: `admin/pages/tools.php`**

Change 1 — Show File Editor entry with permissions message:
```php
// Before:
if ( WPSEO_Utils::allow_system_file_edit() === true && ! is_multisite() ) {
    $tools['file-editor'] = [
        'title' => __( 'File editor', 'wordpress-seo' ),
        'desc'  => __( 'This tool allows you to quickly change important files...', 'wordpress-seo' ),
    ];
}

// After:
if ( ! is_multisite() ) {
    if ( WPSEO_Utils::allow_system_file_edit() === true ) {
        $tools['file-editor'] = [
            'title' => __( 'File editor', 'wordpress-seo' ),
            'desc'  => __( 'This tool allows you to quickly change important files...', 'wordpress-seo' ),
        ];
    }
    else {
        $tools['file-editor'] = [
            'title' => __( 'File editor', 'wordpress-seo' ),
            'desc'  => __( 'You do not have sufficient permissions to edit system files.', 'wordpress-seo' ),
        ];
    }
}
```

Change 2 — Allow page to load for all users:
```php
// Before:
if ( WPSEO_Utils::allow_system_file_edit() === true && ! is_multisite() ) {
    $tool_pages[] = 'file-editor';
}

// After:
if ( ! is_multisite() ) {
    $tool_pages[] = 'file-editor';
}
```

**File 2: `admin/views/tool-file-editor.php`**

Add top-level permission check after `$ht_access_file` is defined:
```php
if ( ! current_user_can( 'edit_files' ) ) {
    echo '<div class="notice notice-error"><p>';
    esc_html_e( 'You do not have sufficient permissions to edit files.', 'wordpress-seo' );
    echo '</p></div>';
    return;
}
```

---

## Pull Request
**PR Link:** [LINK](https://github.com/Yoast/wordpress-seo/pull/23358)
**PR Description:**

*Context:*
Users without the `edit_files` capability who navigate to the Yoast SEO File Editor receive no meaningful feedback. The File Editor link is silently hidden on the Tools page, and if accessed directly, shows a blank page or generic WordPress error. This fix ensures users are clearly informed that they lack the necessary permissions.

*Summary changelog entry:*
Fixes a bug where users without the `edit_files` capability saw a blank page or no explanation when accessing the File Editor, caused by the permission check silently blocking access without displaying an error message.

**Status:** Submitted — Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained
- Navigating a large open source PHP/WordPress plugin codebase
- Understanding WordPress capability system (`current_user_can`, role/capability mapping)
- Using `grep`, `find`, and `sed` to trace bugs through a codebase
- Setting up a local WordPress dev environment with LocalWP
- Building a PHP plugin from source with composer and npm

### Challenges Overcome
- Newer versions of Yoast SEO had removed the File Editor entirely — had to install v18.0 manually
- `npm run build` failed due to lerna config issues — worked around it by copying only the changed PHP files
- The page appeared blank not because of the view file, but because of an upstream gate in `tools.php` — required tracing through multiple files to find the real root cause

### What I'd Do Differently Next Time
- Fork and create the branch before reproducing, not after
- Check the version history of affected files earlier to confirm the feature still exists
- Read `CONTRIBUTING.md` before starting to understand testing requirements upfront

---

## Resources Used
- [Issue #4333](https://github.com/Yoast/wordpress-seo/issues/4333)
- [Yoast SEO GitHub Repository](https://github.com/Yoast/wordpress-seo)
- [WordPress Roles and Capabilities](https://wordpress.org/documentation/article/roles-and-capabilities/)
- [LocalWP](https://localwp.com/)
- [Yoast SEO 18.0 download](https://downloads.wordpress.org/plugin/wordpress-seo.18.0.zip)
