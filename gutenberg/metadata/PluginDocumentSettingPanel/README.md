# How to add metadata to a Gutenvberg PluginDocumentSettingPanel

## Summary
- Register metadata to custom post type
- Create panel
- Link panel to metadata
- Use an image

After days of trying to figure this out, figured I better write it down or forget.  My specific situation wanted to add a second "featured image" to a custom post type.  

I also wanted to take advantage of using the Gutenberg instead of previous PHP versions as I will have a little more flexability and be able to style things nicely.

So lets get started. (I'm going to assume a plugin has already been registered and enabled)

## Registering Stuff

- Register meta fields
- register our gutenberg js file
- Add an editor style

__Add this to your plugins PHP file__

```
function mv_portfolio_sidebar_register() {
  /**
   * Register meta fields
   */
  register_post_meta(               // Note: using register_post_meta instead of register_meta
      'mv_portfolios',              // The custom post type
      'mv_portfolio_before_image',  // The meta field
      array(
        'single' => true,           // It's a single value
        'type' => 'number',         // I've devided to save a number, could be array, string, object, etc.
        'show_in_rest' => true      // Enable this to be returned in rest so that it's accessible, if you are using an array be sure to specify your schema here
    ) 
  );    
  
  /**
   * Register script to control display of sidebar & saving of meta data
   */
	wp_register_script(
    'mv-portfolio-sidebar-js',      // Name of script
    plugins_url( 'portfolio-sidebar.js', __FILE__ ),        // Location of script
        array( 'wp-plugins', 'wp-edit-post', 'wp-element' ),    // What components your script will need
        filemtime( plugin_dir_path( __FILE__ ) . 'portfolio-sidebar.js' ), // location of script (same as above)
    true // Load in footer
  );
  /**
   * Register css to be used in editor
   */
  wp_register_style(
		'sidebar-editor-css',       // Name of style sheet
		plugins_url( 'portfolio-sidebar-editor.css', __FILE__ ) // location of style sheet
  );
  
}
// have this run within an init action so it's available everwhere
add_action( 'init', 'mv_portfolio_sidebar_register' );  

/**
 * Enqueue scripts & styles 
 */
function mv_portfolio_sidebar_script_enqueue() {
  
  // // Only show for porfolio content type
  $screen = get_current_screen();
  if( $screen->post_type != 'mv_portfolios' ) return; 

  // register sidebar
  wp_enqueue_script( 'mv-portfolio-sidebar-js' );  // Referencing name of script
}
add_action( 'enqueue_block_editor_assets', 'mv_portfolio_sidebar_script_enqueue' );

function mv_portfolio_sidebar_style_enqueue() {
	wp_enqueue_style( 'sidebar-editor-css' ); // Referencing name of style sheet
}
add_action( 'enqueue_block_assets', 'mv_portfolio_sidebar_style_enqueue' );
```


## Adding some stylesheets

These are here specifically for the image box, to make it look similar to the "featured image" selector

__portfolio-sidebar-editor.css__
```
.mv-portfolio-panel-content .components-responsive-wrapper{
	position: relative;
    overflow: hidden;
}
.mv-portfolio-panel-content .components-responsive-wrapper>span{
	padding-bottom: 62.5%;
}
.mv-portfolio-panel-content .components-responsive-wrapper>img{
	position: absolute;
	top:-100%; left:0; right: 0; bottom:-100%;
	margin: auto;
}
```


## Complete Document Panel

Here's everything, from accessing to storing meta data & friendly viewing of the preview image.  You can easily add additional control types (be sure to iave a withSelect and withDispatch for each)

__portfolio-sidebar.js__
```
( function( wp ) {
    // Allow plugin to be registered with Gutenberg
	var registerPlugin = wp.plugins.registerPlugin;
    // I want my panel ta appear as a Document setting, but you could change this to almost anything.
	var PluginDocumentSettingPanel = wp.editPost.PluginDocumentSettingPanel;
    // Allows to create HTML elements in panel
	var el = wp.element.createElement;
    // Media upload control
	var MediaUpload = wp.blockEditor.MediaUpload;
    // withSelect used to retrieve metadata (in this scenario)
	var withSelect = wp.data.withSelect;
    // withDispatch used to set metadata (in this scenario)
	var withDispatch = wp.data.withDispatch;
    // Allows you to create custom controls
	var compose = wp.compose.compose;

	
	// Register Meta Media Upload Fields
    // Here I'm composing a control, that connects to metadata
    // This is designed to be re-usable, in the registerPlugin is where we specify what metakey we are accessing
    // You can create these again usina a different "return" to match your own desired component.
	var MetaMediaControl = compose(
		withDispatch( function( dispatch, props ) {
			return {
                // Sets the metadata based on value when setMetaFieldValue is called
				setMetaFieldValue: function( value ) {
					dispatch( 'core/editor' ).editPost(
						{ meta: { [ props.fieldName ]: value } }
					);
				}
			}
		} ),
		withSelect( function( select, props ) {
            // Retreive the meta value
			metaValue = select( 'core/editor' ).getEditedPostAttribute( 'meta' )[ props.fieldName ];
            // I'm adding extra steps here for proper preview of the image
			if(metaValue == 0) metaValue = null;
			return {
				metaFieldValue: metaValue, // Asign the metadata to this value
				metaFieldImageObj: (metaValue? select('core').getMedia(metaValue): null)  // Get the image object to allow for previewing within the panel
			}
		} )
	)( function( props ) {
        // Reference the original control to display the interactive component
		return el( MediaUpload, {
			type: 'image',
			value: props.metaFieldValue,  // Access the metadata refrenced within withSelect
			onSelect: function( content ) {
				props.setMetaFieldValue( content.id ); // Set the metadata using withDispatch
			},
			render: function( obj ) {
                // Create a button so user can click something,
                // I've formatted everything to match the "featured image" selector.
                // I also have a bunch if inline IF statements to allow formatting of  "preview" or  "select image" styling
				return [el(wp.components.Button, {
					className: 'components-button '+(props.metaFieldValue != null? 'editor-post-featured-image__preview':'editor-post-featured-image__toggle'),
					onClick: obj.open  // open the media selector window on click
					},
						(props.metaFieldValue != null?
							// has image
							el('span',{className: 'components-responsive-wrapper'},
								el('span', {}),
								el('img',{
									className: 'components-responsive-wrapper__content',
									src: (props.metaFieldImageObj? props.metaFieldImageObj.source_url : null)
								})	
							)
						:
							// does not have image
							el( 'span', {},'Set before image')	
						) // Endif
					),
					// Has image - display remove button
					(props.metaFieldValue != null?
						el(wp.components.Button,{
							onClick: function(){
								props.setMetaFieldValue(null);
							},
							className: 'components-button is-link is-destructive'
							},
							'Remove before image'
						)
					: null)
				]
			}
		} );
	} );	
    // Register the plugin
    // We've devined everything, now it's time to activate it
	registerPlugin( 
        'mv-portfolio-panel', // Name of my panel
        {
		render: function() {
			return el( PluginDocumentSettingPanel,  // Here's the type of panel I'm using - change this to something else if you need
				{
					name: 'mv-portfolio-panel', // Name of my panel
					icon: 'format-image',       // Icon
					title: 'Before Image',      // Visual title
				},
				el( 'div',  // Let's wrap the content in a div
					{ className: 'mv-portfolio-panel-content' },
					el( MetaMediaControl, // Call my custom control composed above
						{ 
                            fieldName: 'mv_portfolio_before_image'   // The metakey I want access too
                        }
					)
				)
			);
		}
	} );
} )( window.wp );
```