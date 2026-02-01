---
name: wp-apis
description: Use when working with WordPress core APIs in plugins or themes — admin menus, shortcodes, meta boxes, custom post types, taxonomies, HTTP API, WP-Cron, dashboard widgets, users and roles, privacy tools, theme mods, site health, global variables, responsive images, and advanced hooks.
---

# WordPress Core APIs

---

## 1. Administration Menus

### Top-Level Menu

```php
add_action( 'admin_menu', 'map_register_menus' );

function map_register_menus(): void {
    add_menu_page(
        __( 'My Plugin', 'my-plugin' ),   // Page title (<title> tag).
        __( 'My Plugin', 'my-plugin' ),   // Menu label.
        'manage_options',                  // Capability required.
        'my-plugin',                       // Menu slug (unique).
        'map_render_admin_page',           // Callback that outputs page HTML.
        'dashicons-admin-generic',         // Icon URL or dashicon class.
        80                                 // Position (lower = higher).
    );
}
```

### Submenus

```php
add_submenu_page(
    'my-plugin',                         // Parent slug.
    __( 'Settings', 'my-plugin' ),       // Page title.
    __( 'Settings', 'my-plugin' ),       // Menu label.
    'manage_options',                     // Capability.
    'my-plugin-settings',                // Submenu slug.
    'map_render_settings_page'           // Callback.
);
```

| Helper Function             | Parent Page              |
|-----------------------------|--------------------------|
| `add_dashboard_page()`      | Dashboard                |
| `add_posts_page()`          | Posts                    |
| `add_media_page()`          | Media                    |
| `add_pages_page()`          | Pages                    |
| `add_comments_page()`       | Comments                 |
| `add_theme_page()`          | Appearance               |
| `add_plugins_page()`        | Plugins                  |
| `add_users_page()`          | Users                    |
| `add_management_page()`     | Tools                    |
| `add_options_page()`        | Settings                 |

### Form Processing Pattern

Use `load-{$hook_suffix}` to process form submissions before output:

```php
function map_register_menus(): void {
    $hook = add_menu_page( /* ... */ );
    add_action( "load-{$hook}", 'map_handle_form_submit' );
}

function map_handle_form_submit(): void {
    if ( 'POST' !== $_SERVER['REQUEST_METHOD'] ) {
        return;
    }
    check_admin_referer( 'map_save_action', 'map_nonce' );
    wp_safe_redirect( add_query_arg( 'updated', '1', wp_get_referer() ) );
    exit;
}
```

### Removing Menus

```php
remove_menu_page( 'edit-comments.php' );
remove_submenu_page( 'options-general.php', 'options-writing.php' );
```

---

## 2. Shortcodes

### Registration

```php
add_shortcode( 'map_greeting', 'map_greeting_shortcode' );

// Handler: ($atts, $content, $tag) — always RETURN, never echo.
function map_greeting_shortcode( $atts, $content = null, $tag = '' ): string {
    $atts = shortcode_atts(
        array(
            'name'  => 'World',
            'color' => 'blue',
        ),
        $atts,
        $tag  // Enables the shortcode_atts_{$tag} filter for overrides.
    );

    return sprintf(
        '<span style="color:%s">Hello, %s!</span>',
        esc_attr( $atts['color'] ),
        esc_html( $atts['name'] )
    );
}
```

### Self-Closing vs Enclosing

```
[map_greeting name="Alice"]                         <!-- Self-closing -->
[map_box]This is wrapped content.[/map_box]         <!-- Enclosing -->
```

For enclosing shortcodes, call `do_shortcode( $content )` to process nested shortcodes:

```php
function map_box_shortcode( $atts, $content = null ): string {
    return '<div class="map-box">' . do_shortcode( (string) $content ) . '</div>';
}
```

### Rules

- **Return, don't echo** — shortcodes must return HTML, any `echo` corrupts page layout.
- **Prefix names** — use a unique prefix (`map_`) to avoid collisions.
- **Escape output** — all attribute values through `esc_attr()` / `esc_html()`.

---

## 3. Custom Meta Boxes

### Registering

```php
add_action( 'add_meta_boxes', 'map_add_meta_boxes' );

function map_add_meta_boxes(): void {
    add_meta_box(
        'map_details',                      // Unique ID.
        __( 'Extra Details', 'my-plugin' ), // Box title.
        'map_render_details_meta_box',      // Render callback.
        'post',                             // Screen (post type or array of types).
        'side',                             // Context: normal | side | advanced.
        'high'                              // Priority: high | core | default | low.
    );
}
```

### Rendering

```php
function map_render_details_meta_box( WP_Post $post ): void {
    $value = get_post_meta( $post->ID, '_map_detail', true );
    wp_nonce_field( 'map_save_detail', 'map_detail_nonce' );
    printf(
        '<label for="map-detail">%s</label>
         <input type="text" id="map-detail" name="map_detail" value="%s" class="widefat" />',
        esc_html__( 'Detail:', 'my-plugin' ),
        esc_attr( $value )
    );
}
```

### Saving

```php
add_action( 'save_post', 'map_save_detail_meta', 10, 2 );

function map_save_detail_meta( int $post_id, WP_Post $post ): void {
    if ( ! isset( $_POST['map_detail_nonce'] )
        || ! wp_verify_nonce( $_POST['map_detail_nonce'], 'map_save_detail' )
    ) {
        return;
    }
    if ( defined( 'DOING_AUTOSAVE' ) && DOING_AUTOSAVE ) {
        return;
    }
    if ( ! current_user_can( 'edit_post', $post_id ) ) {
        return;
    }
    if ( isset( $_POST['map_detail'] ) ) {
        update_post_meta(
            $post_id,
            '_map_detail',
            sanitize_text_field( wp_unslash( $_POST['map_detail'] ) )
        );
    }
}
```

### OOP Pattern

```php
class MAP_Detail_Meta_Box {
    public static function register(): void {
        add_action( 'add_meta_boxes', array( __CLASS__, 'add' ) );
        add_action( 'save_post', array( __CLASS__, 'save' ), 10, 2 );
    }
    public static function add(): void { /* add_meta_box(...) */ }
    public static function render( WP_Post $post ): void { /* output fields */ }
    public static function save( int $post_id, WP_Post $post ): void { /* verify nonce + save */ }
}
MAP_Detail_Meta_Box::register();
```

### Removing Meta Boxes

```php
remove_meta_box( 'postcustom', 'post', 'normal' );   // Custom Fields.
remove_meta_box( 'slugdiv', 'post', 'normal' );      // Slug.
```

---

## 4. Custom Post Types & Taxonomies

### Registering a Custom Post Type

```php
add_action( 'init', 'map_register_post_types' );

function map_register_post_types(): void {
    register_post_type( 'map_portfolio', array(
        'labels'             => array(
            'name'               => __( 'Portfolio', 'my-plugin' ),
            'singular_name'      => __( 'Project', 'my-plugin' ),
            'add_new_item'       => __( 'Add New Project', 'my-plugin' ),
            'edit_item'          => __( 'Edit Project', 'my-plugin' ),
            'not_found'          => __( 'No projects found.', 'my-plugin' ),
            'not_found_in_trash' => __( 'No projects found in Trash.', 'my-plugin' ),
        ),
        'public'             => true,
        'has_archive'        => true,
        'show_in_rest'       => true,       // Required for Block Editor + REST API.
        'supports'           => array( 'title', 'editor', 'thumbnail', 'excerpt', 'revisions' ),
        'rewrite'            => array( 'slug' => 'portfolio' ),
        'menu_icon'          => 'dashicons-portfolio',
        'capability_type'    => 'post',
    ) );
}
```

### Key `supports` Values

| Value          | Feature Enabled                |
|----------------|--------------------------------|
| `title`        | Title field                    |
| `editor`       | Content editor                 |
| `thumbnail`    | Featured image                 |
| `excerpt`      | Excerpt field                  |
| `revisions`    | Revision history               |
| `page-attributes` | Template + menu order       |
| `custom-fields` | Custom fields meta box        |
| `comments`     | Comments                       |

### Registering a Custom Taxonomy

```php
add_action( 'init', 'map_register_taxonomies' );

function map_register_taxonomies(): void {
    register_taxonomy( 'map_skill', array( 'map_portfolio' ), array(
        'labels'            => array(
            'name'          => __( 'Skills', 'my-plugin' ),
            'singular_name' => __( 'Skill', 'my-plugin' ),
            'add_new_item'  => __( 'Add New Skill', 'my-plugin' ),
        ),
        'hierarchical'      => true,       // true = categories, false = tags.
        'show_in_rest'      => true,       // Block Editor support.
        'rewrite'           => array( 'slug' => 'skill' ),
        'show_admin_column' => true,       // Column in post list table.
    ) );
}
```

### Flushing Rewrite Rules

Rewrite rules are cached. Flush on activation only — never on every request:

```php
register_activation_hook( __FILE__, function (): void {
    map_register_post_types();
    map_register_taxonomies();
    flush_rewrite_rules();
} );
```

### Template Hierarchy for CPTs

| Template File                      | When                        |
|------------------------------------|-----------------------------|
| `single-map_portfolio.php`        | Single CPT post             |
| `archive-map_portfolio.php`       | CPT archive                 |
| `taxonomy-map_skill.php`          | Custom taxonomy archive     |
| `taxonomy-map_skill-{slug}.php`   | Specific term archive       |

---

## 5. HTTP API

### GET Request

```php
$response = wp_remote_get( 'https://api.example.com/data', array(
    'timeout' => 15,
    'headers' => array(
        'Authorization' => 'Bearer ' . $api_key,
        'Accept'        => 'application/json',
    ),
) );

if ( is_wp_error( $response ) ) {
    return new WP_Error( 'api_failure', $response->get_error_message() );
}

$code = wp_remote_retrieve_response_code( $response );
if ( $code < 200 || $code >= 300 ) {
    return new WP_Error( 'api_http_error', "HTTP {$code}" );
}

$data = json_decode( wp_remote_retrieve_body( $response ), true );
```

### POST Request

```php
$response = wp_remote_post( 'https://api.example.com/submit', array(
    'timeout' => 30,
    'body'    => wp_json_encode( array( 'name' => 'Alice' ) ),
    'headers' => array( 'Content-Type' => 'application/json' ),
) );
```

### Default Args

| Argument      | Default        | Notes                                  |
|---------------|----------------|----------------------------------------|
| `method`      | `GET`          | Or `POST`, `PUT`, `DELETE`, etc.       |
| `timeout`     | `5`            | Seconds. Increase for slow APIs.       |
| `redirection` | `5`            | Max redirects to follow.               |
| `httpversion` | `1.0`          | Use `1.1` for keep-alive.             |
| `sslverify`   | `true`         | **Never disable in production.**       |
| `blocking`    | `true`         | `false` = fire-and-forget.             |

### Caching External Requests with Transients

```php
function map_get_external_data(): array {
    $cached = get_transient( 'map_api_data' );
    if ( false !== $cached ) {
        return $cached;
    }

    $response = wp_remote_get( 'https://api.example.com/data' );
    if ( is_wp_error( $response ) ) {
        return array();
    }

    $data = json_decode( wp_remote_retrieve_body( $response ), true );
    if ( ! is_array( $data ) ) {
        return array();
    }

    set_transient( 'map_api_data', $data, HOUR_IN_SECONDS );
    return $data;
}
```

---

## 6. WP-Cron

### Scheduling a Recurring Event

```php
register_activation_hook( __FILE__, 'map_activate_cron' );

function map_activate_cron(): void {
    if ( ! wp_next_scheduled( 'map_daily_cleanup' ) ) {
        wp_schedule_event( time(), 'daily', 'map_daily_cleanup' );
    }
}

add_action( 'map_daily_cleanup', 'map_run_cleanup' );
function map_run_cleanup(): void { /* daily maintenance */ }
```

### Single Event

```php
wp_schedule_single_event( time() + 300, 'map_deferred_task', array( $post_id ) );
add_action( 'map_deferred_task', function ( int $post_id ): void { /* runs once */ } );
```

### Custom Intervals

```php
add_filter( 'cron_schedules', function ( array $schedules ): array {
    $schedules['map_five_minutes'] = array(
        'interval' => 300,
        'display'  => __( 'Every 5 Minutes', 'my-plugin' ),
    );
    return $schedules;
} );
```

### Cleanup on Deactivation

```php
register_deactivation_hook( __FILE__, function (): void {
    $timestamp = wp_next_scheduled( 'map_daily_cleanup' );
    if ( $timestamp ) {
        wp_unschedule_event( $timestamp, 'map_daily_cleanup' );
    }
} );
```

### Real Cron for Production

WP-Cron is triggered by page visits — unreliable on low-traffic sites:

```php
// wp-config.php — disable virtual cron.
define( 'DISABLE_WP_CRON', true );
```

```bash
# System crontab — trigger WP-Cron every minute via WP-CLI.
* * * * * cd /var/www/html && wp cron event run --due-now > /dev/null 2>&1
```

---

## 7. Dashboard Widgets

```php
add_action( 'wp_dashboard_setup', 'map_add_dashboard_widget' );

function map_add_dashboard_widget(): void {
    wp_add_dashboard_widget(
        'map_status_widget',                     // Widget ID.
        __( 'Plugin Status', 'my-plugin' ),      // Title.
        'map_render_status_widget'                // Render callback.
    );
}

function map_render_status_widget(): void {
    $count = wp_count_posts( 'map_portfolio' );
    printf(
        '<p>%s</p>',
        sprintf( esc_html__( 'Published projects: %d', 'my-plugin' ), absint( $count->publish ) )
    );
}
```

For side column placement, use `add_meta_box()` with `'dashboard'` screen and `'side'` context.

### Removing Default Widgets

```php
add_action( 'wp_dashboard_setup', function (): void {
    remove_meta_box( 'dashboard_quick_press', 'dashboard', 'side' );
    remove_meta_box( 'dashboard_primary', 'dashboard', 'side' );
} );
```

---

## 8. Users & Roles

### Adding a Custom Role

```php
// Run once on activation — roles persist in the database.
register_activation_hook( __FILE__, function (): void {
    add_role( 'map_project_manager', __( 'Project Manager', 'my-plugin' ), array(
        'read'         => true,
        'edit_posts'   => true,
        'delete_posts' => true,
        'publish_posts' => true,
        'upload_files'  => true,
    ) );
} );

// Clean up on uninstall.
register_uninstall_hook( __FILE__, function (): void {
    remove_role( 'map_project_manager' );
} );
```

### Custom Capabilities

```php
// Add on activation:
$admin = get_role( 'administrator' );
if ( $admin ) {
    $admin->add_cap( 'manage_map_settings' );
}

// Check in code:
if ( current_user_can( 'manage_map_settings' ) ) { /* admin-only UI */ }
```

### User Meta

```php
update_user_meta( $user_id, '_map_onboarded', '1' );           // Store.
$val = get_user_meta( $user_id, '_map_onboarded', true );       // Retrieve.
delete_user_meta( $user_id, '_map_onboarded' );                 // Delete.
```

### Querying Users

```php
$query = new WP_User_Query( array(
    'role'       => 'map_project_manager',
    'orderby'    => 'registered',
    'order'      => 'DESC',
    'number'     => 20,
    'meta_query' => array(                    // phpcs:ignore WordPress.DB.SlowDBQuery
        array( 'key' => '_map_onboarded', 'value' => '1', 'compare' => '=' ),
    ),
) );

foreach ( $query->get_results() as $user ) { /* WP_User */ }
```

---

## 9. Privacy API

WordPress 4.9.6+ includes GDPR/privacy tools. Plugins storing personal data
should register exporters and erasers.

### Personal Data Exporter

```php
add_filter( 'wp_privacy_personal_data_exporters', function ( array $exporters ): array {
    $exporters['my-plugin'] = array(
        'exporter_friendly_name' => __( 'My Plugin Data', 'my-plugin' ),
        'callback'               => 'map_privacy_exporter',
    );
    return $exporters;
} );

function map_privacy_exporter( string $email, int $page = 1 ): array {
    $user = get_user_by( 'email', $email );
    $data = array();

    if ( $user ) {
        $pref = get_user_meta( $user->ID, '_map_preference', true );
        if ( $pref ) {
            $data[] = array(
                'group_id'    => 'map-preferences',
                'group_label' => __( 'Plugin Preferences', 'my-plugin' ),
                'item_id'     => "map-pref-{$user->ID}",
                'data'        => array(
                    array( 'name' => __( 'Preference', 'my-plugin' ), 'value' => $pref ),
                ),
            );
        }
    }

    return array( 'data' => $data, 'done' => true );
    // Set 'done' => false and increment $page for large datasets.
}
```

### Personal Data Eraser

Same pattern — register via `wp_privacy_personal_data_erasers` filter. Callback returns:

```php
return array(
    'items_removed'  => $count,    // How many items were erased.
    'items_retained' => 0,         // Items kept (with reason in 'messages').
    'messages'       => array(),   // Human-readable notes.
    'done'           => true,      // false + $page for pagination.
);
```

### Privacy Policy Suggestion

```php
add_action( 'admin_init', function (): void {
    if ( function_exists( 'wp_add_privacy_policy_content' ) ) {
        wp_add_privacy_policy_content(
            __( 'My Plugin', 'my-plugin' ),
            wp_kses_post( __( '<p>This plugin stores your preferences.</p>', 'my-plugin' ) )
        );
    }
} );
```

---

## 10. Theme Modification API

Theme mods are settings scoped to the active theme. Unlike options (global), they
travel with the theme — switching themes resets to that theme's stored mods.

### Core Functions

```php
// Store a mod for the current theme.
set_theme_mod( 'header_color', '#ff6600' );

// Retrieve with a fallback default.
$color = get_theme_mod( 'header_color', '#000000' );

// Remove a single mod.
remove_theme_mod( 'header_color' );

// Retrieve all mods for the active theme (returns array or empty array).
$all_mods = get_theme_mods();
```

### Customizer Integration

Theme mods are the standard storage backend for the Customizer:

```php
add_action( 'customize_register', function ( WP_Customize_Manager $wp_customize ): void {
    $wp_customize->add_setting( 'map_accent_color', array(
        'default'           => '#0073aa',
        'type'              => 'theme_mod',        // Default — stored per-theme.
        'sanitize_callback' => 'sanitize_hex_color',
        'transport'         => 'postMessage',       // Live preview via JS.
    ) );

    $wp_customize->add_control( new WP_Customize_Color_Control(
        $wp_customize,
        'map_accent_color',
        array(
            'label'   => __( 'Accent Color', 'my-theme' ),
            'section' => 'colors',
        )
    ) );
} );
```

### Using Theme Mods in Templates

```php
<style>
    .site-header { background-color: <?php echo esc_attr( get_theme_mod( 'header_color', '#fff' ) ); ?>; }
</style>
```

### Theme Mod vs Option

| Feature             | `theme_mod`                      | `option`                   |
|---------------------|----------------------------------|----------------------------|
| Scope               | Per-theme                        | Global (all themes)        |
| Switching themes    | Reverts to new theme's values    | Persists                   |
| Storage             | Single `theme_mods_{$theme}` row | Individual option rows     |
| Customizer default  | Yes (`type => 'theme_mod'`)      | Must set `type => 'option'`|
| Filter              | `theme_mod_{$name}`              | `option_{$name}`           |

Use `theme_mod` for visual/display settings. Use `option` for functional plugin settings.

---

## 11. Site Health API

The Site Health screen (Tools → Site Health) can be extended with custom tabs and
debug information sections.

### Custom Tab

```php
add_filter( 'site_health_navigation_tabs', function ( array $tabs ): array {
    $tabs['map-status'] = esc_html__( 'Plugin Status', 'my-plugin' );
    return $tabs;
} );

add_action( 'site_health_tab_content', function ( string $tab ): void {
    if ( 'map-status' !== $tab ) {
        return;
    }
    echo '<div class="health-check-body">';
    // Render tab content.
    echo '</div>';
} );
```

### Custom Status Test

```php
add_filter( 'site_status_tests', function ( array $tests ): array {
    $tests['direct']['map_check'] = array(
        'label' => __( 'My Plugin Check', 'my-plugin' ),
        'test'  => 'map_site_health_test',
    );
    return $tests;
} );

function map_site_health_test(): array {
    $result = array(
        'label'       => __( 'My Plugin is configured', 'my-plugin' ),
        'status'      => 'good',             // good | recommended | critical.
        'badge'       => array(
            'label' => __( 'Performance', 'my-plugin' ),
            'color' => 'blue',               // blue | green | red | orange | purple | gray.
        ),
        'description' => '<p>' . __( 'Everything looks good.', 'my-plugin' ) . '</p>',
        'actions'     => '',                  // HTML for action links.
        'test'        => 'map_check',         // Must match the key above.
    );

    if ( ! get_option( 'map_api_key' ) ) {
        $result['status']      = 'recommended';
        $result['label']       = __( 'My Plugin API key is missing', 'my-plugin' );
        $result['description'] = '<p>' . __( 'Add an API key for full functionality.', 'my-plugin' ) . '</p>';
    }

    return $result;
}
```

### Debug Information Section

```php
add_filter( 'debug_information', function ( array $info ): array {
    $info['my-plugin'] = array(
        'label'  => __( 'My Plugin', 'my-plugin' ),
        'fields' => array(
            'version' => array(
                'label' => __( 'Version', 'my-plugin' ),
                'value' => MAP_VERSION,
            ),
            'api_connected' => array(
                'label' => __( 'API Connected', 'my-plugin' ),
                'value' => get_option( 'map_api_key' ) ? __( 'Yes' ) : __( 'No' ),
            ),
        ),
    );
    return $info;
} );
```

---

## 12. Global Variables

WordPress uses global variables throughout its codebase. Prefer API functions
when available; use globals only when no function exists.

### Access Pattern

```php
global $post;
// Now $post is available as WP_Post|null.
```

### Inside the Loop

| Variable        | Type       | Description                              |
|-----------------|------------|------------------------------------------|
| `$post`         | `WP_Post`  | Current post being processed             |
| `$authordata`   | `WP_User`  | Current post's author                    |
| `$page`         | `int`      | Current page of a paginated post         |
| `$multipage`    | `bool`     | Whether post has `<!--nextpage-->` splits|
| `$numpages`     | `int`      | Total pages in paginated post            |
| `$more`         | `bool`     | Whether to show content past `<!--more-->`|

### Core Objects

| Variable          | Type          | Preferred API                          |
|-------------------|---------------|----------------------------------------|
| `$wpdb`           | `wpdb`        | No alternative — use directly          |
| `$wp_query`       | `WP_Query`    | Template tags: `have_posts()`, `the_post()` |
| `$wp_rewrite`     | `WP_Rewrite`  | `get_option('permalink_structure')`    |
| `$wp_roles`       | `WP_Roles`    | `wp_roles()`, `current_user_can()`     |
| `$wp_admin_bar`   | `WP_Admin_Bar`| `add_action('admin_bar_menu', ...)`    |
| `$wp_locale`      | `WP_Locale`   | `wp_date()`, `date_i18n()`            |

### Admin & Version

| Variable          | Type     | Description                            |
|-------------------|----------|----------------------------------------|
| `$pagenow`        | `string` | Current admin page filename            |
| `$post_type`      | `string` | Current post type in admin screens     |
| `$wp_version`     | `string` | WordPress version number               |
| `$wp_db_version`  | `int`    | Database schema version                |

### Server Detection

| Variable      | Type   | True when...        |
|---------------|--------|---------------------|
| `$is_apache`  | `bool` | Running on Apache   |
| `$is_nginx`   | `bool` | Running on Nginx    |
| `$is_IIS`     | `bool` | Running on IIS      |

### Best Practices

- **Use API functions first** — `get_queried_object()` instead of `global $wp_query; $wp_query->get_queried_object()`.
- **Always declare `global`** before use: `global $wpdb;`
- **Never modify** `$post` without `wp_reset_postdata()` afterward.
- **`$wpdb` is the exception** — it has no wrapper and must be used directly for custom queries.

---

## 13. Responsive Images

WordPress 4.4+ automatically adds `srcset` and `sizes` attributes to images,
letting browsers pick the optimal file for the viewport and pixel density.

### Default Behavior

When you use `wp_get_attachment_image()` or insert images in the editor, WordPress
generates markup like:

```html
<img src="image-768x512.jpg"
     srcset="image-300x200.jpg 300w, image-768x512.jpg 768w, image-1024x683.jpg 1024w"
     sizes="(max-width: 768px) 100vw, 768px"
     alt="...">
```

### Key Functions

```php
// Full <img> tag with srcset/sizes.
echo wp_get_attachment_image( $attachment_id, 'medium' );

// Just the srcset value string.
$srcset = wp_get_attachment_image_srcset( $attachment_id, 'medium' );

// Just the sizes value string.
$sizes = wp_get_attachment_image_sizes( $attachment_id, 'medium' );
```

### Custom Image Sizes

```php
add_action( 'after_setup_theme', function (): void {
    add_image_size( 'hero', 1200, 600, true );      // Hard crop.
    add_image_size( 'card', 400, 300, true );
} );
```

New sizes are included in `srcset` automatically if they match the original's aspect ratio.

### Customizing `sizes` Attribute

The default `sizes` assumes full-width (`100vw` up to image width). Override for your layout:

```php
add_filter( 'wp_calculate_image_sizes', function ( string $sizes, array $size, ?string $image_src, ?array $image_meta, int $attachment_id ): string {
    // Two-column layout: images are 50% wide above 768px.
    return '(max-width: 768px) 100vw, 50vw';
}, 10, 5 );
```

### Customizing `srcset` Sources

```php
add_filter( 'wp_calculate_image_srcset', function ( array $sources, array $size_array, string $image_src, array $image_meta, int $attachment_id ): array {
    // Remove sources wider than 1600px.
    foreach ( $sources as $width => $source ) {
        if ( $width > 1600 ) {
            unset( $sources[ $width ] );
        }
    }
    return $sources;
}, 10, 5 );
```

### Max Width

The `max_srcset_image_width` filter caps which sizes appear in `srcset` (default: 2048px):

```php
add_filter( 'max_srcset_image_width', function (): int {
    return 1600; // Don't include anything wider than 1600px.
} );
```

### Lazy Loading

WordPress 5.5+ adds `loading="lazy"` to images and iframes automatically.
Override per-image via the `wp_img_tag_add_loading_optimization_attrs` filter or
by passing `'loading' => 'eager'` to `wp_get_attachment_image()` for above-the-fold images.

---

## 14. Advanced Hooks

### Removing Actions & Filters

Removal must happen after the original `add_action()` has run and at the same priority:

```php
add_action( 'plugins_loaded', function (): void {
    remove_action( 'wp_head', 'wp_generator' );
    remove_filter( 'the_content', 'wpautop' );
} );
```

For class-based hooks, you need the exact instance or use the `$wp_filter` global (fragile).

### Single-Run Guards

```php
add_action( 'save_post', function ( int $post_id ): void {
    if ( did_action( 'save_post' ) > 1 ) {
        return;     // Prevent recursive triggers.
    }
} );
```

### Inspection Functions

| Function             | Returns                                          |
|----------------------|--------------------------------------------------|
| `did_action()`       | Number of times an action has fired (int).       |
| `doing_action()`     | Whether a specific action is currently running.  |
| `doing_filter()`     | Whether a specific filter is currently running.  |
| `current_action()`   | Name of the action currently being executed.     |
| `current_filter()`   | Name of the current filter (works for actions).  |
| `has_action()`       | Whether a callback is registered for an action.  |
| `has_filter()`       | Whether a callback is registered for a filter.   |

### Making Your Plugin Extensible

```php
// Action — let others react to events.
do_action( 'map_order_processed', $order );

// Filter — let others modify data.
$name = apply_filters( 'map_display_name', $name, $user_id );
```

Rules: prefix all hook names, document with `@since`/`@param`, pass enough context, never change signatures after release.

---

## 15. Common Mistakes

| Mistake | Fix |
|---------|-----|
| Flushing rewrite rules on every request | Only flush on activation: `register_activation_hook` + `flush_rewrite_rules()` |
| Registering CPTs/taxonomies in `admin_init` | Use `init` — they must be available on the frontend too |
| `echo` inside a shortcode handler | Always `return` the HTML string |
| Saving meta without nonce/capability check | Verify nonce, check `DOING_AUTOSAVE`, check `current_user_can()` |
| Scheduling cron without `wp_next_scheduled()` guard | Always check first or you create duplicate events |
| Not cleaning up cron on deactivation | `wp_unschedule_event()` in `register_deactivation_hook` |
| Using `$wpdb->query()` for HTTP API work | Use `wp_remote_get()` / `wp_remote_post()` |
| Disabling `sslverify` in production | Only disable for local development; never in production |
| Adding roles/caps on every page load | Roles persist in DB — add on activation, remove on uninstall |
| Forgetting `show_in_rest => true` on CPTs | Required for Block Editor and REST API access |
| Not paginating privacy exporters/erasers | Set `done => false` and use `$page` parameter |
| Calling `remove_action()` too early | Must run after the original `add_action()` and match priority |
| Using `get_option()` for theme-specific settings | Use `get_theme_mod()` — values travel with the theme |
| Modifying `$post` global without reset | Always call `wp_reset_postdata()` after custom loops |
| Hardcoding image sizes in `srcset` | Use `wp_calculate_image_srcset` filter instead |
