# WordPress 7.0+ Block Editor & New APIs Reference

> Load this file for block development (PHP-only and full JS), Interactivity API,
> DataViews/DataForm integration, Abilities API, WP AI Client SDK patterns, and
> the Connectors API.

---

## PHP-Only Block Registration (WordPress 7.0+)

The biggest DX improvement in 7.0 — no JavaScript, no `npm`, no build step:

```php
// register_block_type() now auto-generates Inspector controls for common
// attribute types when 'render_callback' is provided and no editor script is set

add_action( 'init', function(): void {
    register_block_type( 'my-plugin/article-card', [
        'render_callback' => 'my_plugin_render_article_card',
        'attributes'      => [
            // WordPress 7.0 auto-generates UI controls for these types:
            'postId'      => [ 'type' => 'integer', 'default' => 0 ],        // → PostControl
            'showExcerpt' => [ 'type' => 'boolean', 'default' => true ],     // → ToggleControl
            'imageSize'   => [                                                 // → SelectControl
                'type'    => 'string',
                'default' => 'medium',
                'enum'    => [ 'thumbnail', 'medium', 'large', 'full' ],
            ],
            'accentColor' => [ 'type' => 'string', 'default' => '#000000' ], // → ColorPicker
            'maxWords'    => [                                                 // → RangeControl
                'type'    => 'integer',
                'default' => 50,
                'minimum' => 10,
                'maximum' => 200,
            ],
        ],
        'supports' => [
            'html'  => false,
            'align' => [ 'wide', 'full' ],
            'color' => [ 'background' => true, 'text' => true ],
        ],
    ] );
} );

function my_plugin_render_article_card( array $attributes, string $content, WP_Block $block ): string {
    $post = get_post( absint( $attributes['postId'] ) );
    if ( ! $post instanceof WP_Post || 'publish' !== $post->post_status ) {
        return '';
    }

    $wrapper_attrs = get_block_wrapper_attributes( [
        'class' => 'my-plugin-article-card',
    ] );

    $title   = esc_html( get_the_title( $post ) );
    $excerpt = $attributes['showExcerpt']
        ? '<p>' . esc_html( wp_trim_words( $post->post_excerpt ?: $post->post_content, absint( $attributes['maxWords'] ) ) ) . '</p>'
        : '';

    return "<article {$wrapper_attrs}><h2>{$title}</h2>{$excerpt}</article>";
}
```

---

## Full Block with JavaScript (When You Need It)

For blocks requiring interactive editing UIs:

```
my-plugin/
└── src/
    └── blocks/
        └── search-filter/
            ├── block.json
            ├── edit.js
            ├── save.js
            └── index.js
```

```json
// block.json
{
    "$schema": "https://schemas.wp.org/trunk/block.json",
    "apiVersion": 3,
    "name": "my-plugin/search-filter",
    "version": "1.0.0",
    "title": "Search Filter",
    "category": "widgets",
    "editorScript": "file:./index.js",
    "render": "file:./render.php",
    "attributes": {
        "placeholder": { "type": "string", "default": "Search..." },
        "showButton":  { "type": "boolean", "default": true }
    },
    "supports": {
        "html": false,
        "interactivity": true
    }
}
```

```php
// render.php — server-side render for dynamic blocks
<?php
$wrapper_attrs = get_block_wrapper_attributes();
$placeholder   = esc_attr( $attributes['placeholder'] ?? 'Search...' );
$nonce         = wp_create_nonce( 'search_filter_nonce' );
?>
<div <?php echo $wrapper_attrs; ?>
     data-wp-interactive="my-plugin/search-filter"
     data-wp-context='<?php echo esc_attr( wp_json_encode( [ 'query' => '', 'nonce' => $nonce ] ) ); ?>'>
    <input type="search"
           placeholder="<?php echo $placeholder; ?>"
           data-wp-bind:value="context.query"
           data-wp-on:input="actions.onInput" />
</div>
```

---

## Interactivity API (WordPress 7.0+)

The Interactivity API router changed in 7.0 — update `navigate()` calls:

```javascript
// BREAKING CHANGE in 7.0: router API updated
// OLD (pre-7.0):
import { navigate } from '@wordpress/interactivity/router';
navigate( '/new-url' );

// NEW (7.0+):
import { getElement, store } from '@wordpress/interactivity';
const { actions } = store( 'my-plugin/search-filter', {
    state: {
        query: '',
        results: [],
        loading: false,
    },
    actions: {
        onInput( event ) {
            const { state } = getElement();
            state.query = event.target.value;
            actions.fetchResults();
        },
        async fetchResults() {
            const { state } = getElement();
            if ( ! state.query ) {
                state.results = [];
                return;
            }
            state.loading = true;
            try {
                const response = await fetch(
                    `/wp-json/my-plugin/v1/search?q=${encodeURIComponent( state.query )}`,
                    { headers: { 'X-WP-Nonce': state.nonce } }
                );
                state.results = await response.json();
            } finally {
                state.loading = false;
            }
        },
    },
} );
```

---

## Iframed Block Editor (WordPress 7.0 Breaking Change)

The editor is now **always** iframed. This breaks any code that:

- Accesses `window.parent` from inside editor scripts
- Injects styles via `wp_enqueue_style()` expecting them to apply inside the editor
- Uses `document.querySelector()` targeting the outer admin chrome

```javascript
// OLD (broken in 7.0): accessing outer frame from editor script
window.parent.document.querySelector( '.my-admin-bar-item' );

// NEW: use postMessage for cross-frame communication
// In your editor script (inside the iframe):
window.parent.postMessage(
    { type: 'MY_PLUGIN_ACTION', payload: { data: 'value' } },
    window.location.origin   // validate origin — never use '*'
);

// In your admin page script (outside the iframe):
window.addEventListener( 'message', ( event ) => {
    if ( event.origin !== window.location.origin ) return;  // always validate
    if ( event.data?.type !== 'MY_PLUGIN_ACTION' ) return;
    // handle the message
} );
```

```php
// Enqueue editor styles correctly for the iframed editor
add_action( 'enqueue_block_editor_assets', function(): void {
    // This goes inside the editor iframe — correct
    wp_enqueue_style(
        'my-plugin-editor',
        MY_PLUGIN_URL . 'assets/css/editor.css',
        [ 'wp-edit-blocks' ],
        MY_PLUGIN_VERSION
    );
} );

// For styles that need to target BOTH editor and front-end:
add_action( 'init', function(): void {
    register_block_type( 'my-plugin/my-block', [
        'style'        => 'my-plugin-block-style',   // front-end + editor
        'editor_style' => 'my-plugin-editor-only',   // editor only (inside iframe)
    ] );
} );
```

---

## Block Bindings API (WordPress 7.0+)

Connect any custom block attribute to a dynamic data source:

```php
add_action( 'init', function(): void {
    register_block_bindings_source( 'my-plugin/post-field', [
        'label'              => __( 'Post Field', 'my-plugin' ),
        'get_value_callback' => function( array $source_args, WP_Block $block ): mixed {
            $field = sanitize_key( $source_args['field'] ?? '' );
            $post  = $block->context['postId'] ?? get_the_ID();

            return match ( $field ) {
                'title'    => get_the_title( $post ),
                'date'     => get_the_date( 'Y-m-d', $post ),
                'author'   => get_the_author_meta( 'display_name', (int) get_post_field( 'post_author', $post ) ),
                'score'    => (float) get_post_meta( $post, '_score', true ),
                default    => null,
            };
        },
    ] );
} );
```

---

## WP AI Client (WordPress 7.0+)

```php
// wp_ai_client_prompt() — provider-agnostic, routes through Settings > Connectors
// Users configure their own AI provider; your plugin never holds API keys

// Basic usage
$response = wp_ai_client_prompt( 'Summarize this in one sentence: ' . $content );

// With options
$response = wp_ai_client_prompt(
    $prompt,
    [
        'system'      => 'You are a helpful WordPress content assistant.',
        'max_tokens'  => 200,
        'temperature' => 0.7,
    ]
);

if ( is_wp_error( $response ) ) {
    // Handle gracefully — user may not have configured a provider
    return;
}

$text = sanitize_textarea_field( $response['text'] ?? '' );

// Abilities API — expose WordPress actions to AI agents
add_action( 'rest_api_init', function(): void {
    // Register an "ability" that AI agents can invoke
    // Treat this exactly like a REST endpoint — always validate permissions
    register_wp_ability( 'my-plugin/create-draft', [
        'description'         => 'Create a draft post with given title and content.',
        'permission_callback' => fn() => current_user_can( 'edit_posts' ),
        'parameters'          => [
            'title'   => [ 'type' => 'string', 'required' => true ],
            'content' => [ 'type' => 'string', 'required' => true ],
        ],
        'callback' => function( array $params ): array {
            $post_id = wp_insert_post( [
                'post_title'   => sanitize_text_field( $params['title'] ),
                'post_content' => wp_kses_post( $params['content'] ),
                'post_status'  => 'draft',
            ], true );

            if ( is_wp_error( $post_id ) ) {
                return [ 'success' => false, 'error' => $post_id->get_error_message() ];
            }

            return [ 'success' => true, 'post_id' => $post_id ];
        },
    ] );
} );

// Connectors API — read third-party credentials safely
// Never ask users to paste API keys into your plugin's settings
$connector = get_wp_connector( 'openai' );  // reads from Settings > Connectors
if ( $connector && ! is_wp_error( $connector ) ) {
    $api_key = $connector->get_credential( 'api_key' );
}
```

---

## DataViews Integration (WordPress 7.0+)

DataViews replaces `WP_List_Table` on core screens. For custom admin screens,
use the DataViews React component rather than building your own list tables:

```javascript
// Register a custom DataViews-powered admin page
// In your enqueue:
import { DataViews } from '@wordpress/dataviews';
import { useState } from '@wordpress/element';

function MyPluginDataView() {
    const [ view, setView ] = useState( {
        type: 'table',
        perPage: 20,
        page: 1,
        sort: { field: 'date', direction: 'desc' },
    } );

    const { data, isLoading } = useSelect( ( select ) => {
        // Fetch from your REST endpoint
        return select( 'core' ).getEntityRecords( 'postType', 'my_cpt', {
            per_page: view.perPage,
            page: view.page,
            orderby: view.sort.field,
            order: view.sort.direction,
        } );
    } );

    const fields = [
        { id: 'title',  label: 'Title',  enableSorting: true },
        { id: 'date',   label: 'Date',   enableSorting: true },
        { id: 'status', label: 'Status', enableSorting: false },
    ];

    return (
        <DataViews
            data={ data ?? [] }
            fields={ fields }
            view={ view }
            onChangeView={ setView }
            isLoading={ isLoading }
            paginationInfo={ { totalItems: 100, totalPages: 5 } }
            supportedLayouts={ [ 'table', 'grid' ] }
        />
    );
}
```

```php
// For PHP-side: if your plugin customized WP_List_Table on Posts/Pages/Media,
// those hooks still fire but are NOT displayed in DataViews screens.
// You need to register your columns via the REST API and DataViews JS components.

// Transition approach: keep legacy hooks for backwards compat,
// AND register REST fields for DataViews
add_action( 'rest_api_init', function(): void {
    register_rest_field( 'post', 'my_score', [
        'get_callback' => fn( $post ) => (float) get_post_meta( $post['id'], '_score', true ),
        'schema'       => [ 'type' => 'number', 'description' => 'Post quality score.' ],
    ] );
} );
```
*Last Updated: 2026-07-28*
