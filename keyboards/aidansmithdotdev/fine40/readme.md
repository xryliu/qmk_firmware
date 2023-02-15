# Fine!40 PCB (For Mochi40)

![aidansmithdotdev/fine40](https://i.imgur.com/2JMorvxh.png)

The PCB for the Mochi40, a spiritual successor to the unreleased Whimsy recreated from scratch and made completely open source! With an OLED, Rotary Encoder, and headers for both the Elite-C and Nice!Nano, this board gives you all you could want in a 40%.

* Keyboard Maintainer: [Aidan Smith](https://github.com/Aidan-OS)
* Hardware Supported: Fine!40
* Hardware Availability: https://github.com/Aidan-OS/Mochi40

Make example for this keyboard (after setting up your build environment):

    make aidansmithdotdev/fine40:default

Flashing example for this keyboard:

    make aidansmithdotdev/fine40:default:flash

See the [build environment setup](https://docs.qmk.fm/#/getting_started_build_tools) and the [make instructions](https://docs.qmk.fm/#/getting_started_make_guide) for more information. Brand new to QMK? Start with our [Complete Newbs Guide](https://docs.qmk.fm/#/newbs).

## Bootloader

Enter the bootloader in 3 ways:

* **Bootmagic reset**: Hold down the key at (0,0) in the matrix (top left key) and plug in the keyboard
* **Physical reset button**: Briefly press the button on the back of the PCB - some may have pads you must short instead
* **Keycode in layout**: Press the key mapped to `QK_BOOT` if it is available


## OLED instructions

* **Simple logo upload**

1.create a black and white image that is 128 pixels wide and less than 64 pixels tall in something like MS paint (you can even download from online then just resize the image to 128 pixels) 

2.Go to https://joric.github.io/qle/ then use upload image funtion and it will generate hex code for each pixel to show new image in OLED

3.Open fine40.c and replace the hex code in static const char PROGMEM mochi_logo[] = {... } with new ones from previous step

4.Save the file,compile using QMK MSYS, then flash to your mochi

* **Explanation of funtions**

-For 128x64 OLED the OLED_ROTATION_180 function rotates the images to correct orientation, for 128x32, you can change this to OLED_ROTATION_270 so texts are rotated another 90 degress

-config.h has a line #define OLED_DISPLAY_128X64 that tells QMK that screen is 128x64, and for 128x32 comment this line out by pulling 2 forward slash in front will make everything appear more proportional on the screen (//#define OLED_DISPLAY_128X64)

-PROGMEM mochi_logo[] is a arrary that stores the image at pixel level, and for animations you need to make this a 2D array to store multiple picture (will be discussed in more detail in the Advanded functions below)

-switch (get_highest_layer(layer_state))   this block of code identifies the layer that is currently active, text used in Case (i.e. case _MAIN) need to match the text in  enum keyboard_layers defined at top of fine40.c, but actual text desplayed can be changed to any thing you want (i.e  oled_write_ln_P(PSTR("MAIN"), false); can be changed to  oled_write_ln_P(PSTR("BASE"), false);). Other status indications are also possibe and will be disucussed below

* **More Advance Funtions**

Animating the Logo:
These are based on code published by pedker (https://github.com/pedker/OLED-BongoCat-Revision) and for demo purpose I am using pedker's bongo cat pictures in fine40 - animated.c file 

Variables needed: 

        #define IDLE_FRAMES 2 <-this variable indicates the total number of frames in your animation, the more you add the bigger the compiled fiel will be
        
        uint8_t current_idle_frame = 0; <-this variable help draw the frames in sequential order

        #define ANIM_FRAME_DURATION 75 <- if you want to run the animation automatically this variable defines how fast in milisecounds the the pictures changes 
        
        uint32_t anim_timer = 0; <- and this variable helps track time

the picture array can be made 2D using 2 square brackets, the first bracket should say IDLE_FRAMES which specify number of pictures, secound square spefify the number of pixels in each image, yoiu get this number by multipling the width x height of the image (I am using 128x32 in the demo which equals to 4096)
         
         mochi_logo[IDLE_FRAMES][4096]= {
                {hex code for picture 1 },
                {hex code for picture 2 }
                etc,etc
                };

 because it is now a 2 D array, when draw teh image you can sepcify which fram to draw so oled_write_raw_P(mochi_logo, sizeof(mochi_logo)); need to be changed to oled_write_raw_P(mochi_logo[current_idle_frame], sizeof(mochi_logo));  using current_idle_frame as a tracker cycle through the images
 
 if you want animate to move by itself, below code will update the current_idle_frame every x secounds, just put in the render_mochi(void) after the oled_write_raw_P line
 
         if (timer_elapsed32(anim_timer) > ANIM_FRAME_DURATION)
            {
                current_idle_frame = (current_idle_frame + 1) % IDLE_FRAMES;
                anim_timer = timer_read32();
            }
            
 if you want to animation to change only after you press the key, you can add below code as a new function that will only increase teh frame number on key press (fine40.c currently has this code)
 
         bool process_record_user(uint16_t keycode, keyrecord_t *record) {
            if (record->event.pressed) {
              current_idle_frame = (current_idle_frame + 1) % IDLE_FRAMES;
            }
            return true;
          }
  
  
  
  Other Status indicators:
  
  please see fine40 -128x32.c to see how they are used
  
  this code broke will highlight the modifier word that is currenly pressed
  
              void render_mod_status(uint8_t modifiers) {
               //    oled_write_ln_P(PSTR("-----"), false);
                oled_write_ln_P(PSTR("SHFT"), (modifiers & MOD_MASK_SHIFT));
                oled_write_ln_P(PSTR("ALT"), (modifiers & MOD_MASK_ALT));
                oled_write_ln_P(PSTR("CTRL"), (modifiers & MOD_MASK_CTRL));
                oled_write_ln_P(PSTR("GUI"), (modifiers & MOD_MASK_GUI));
            }


this block of code with highligh a chacter when cap lock, num lock or scroll locked in pressed

            void render_keylock_status(led_t led_state) {

                oled_write_P(PSTR("A"), led_state.num_lock);
             oled_write_P(PSTR("1"), led_state.scroll_lock);

                oled_write_P(PSTR("@"), led_state.caps_lock);

            }

the ordering you have in oled_task_user(void) function is the order each of the indicators will display, example below, oled_set_cursor(0, X); will force the stats to appear in the line specified (each line take 8 pixels in height so 128x32 and dispay 16 lines total because it is vertical and 128x64 only 8 lines because it is horizontal)

one issue/bug I did encounter is that when doing the animated logo, only 1 line after the logo is displaced, so I currently have render_mochi() as the last function I call or only the first status indicator works

	oled_set_cursor(0, 0); 
render_layer_status();
oled_set_cursor(0, 1); 
render_mod_status(get_mods()|get_oneshot_mods());

	//render_keylock_status(host_keyboard_led_state());
oled_set_cursor(0, 5); 
	render_mochi();
	

	


