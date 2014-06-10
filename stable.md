# ui
--
    import "github.com/andlabs/ui"

Package ui is a simple package which provides a way to write portable GUI
programs quickly and easily. It aims to run on as many systems as Go itself, but
presently has support for Windows, Mac OS X, and other Unix systems using the
Windows API, Cocoa, and GTK+ 3, respectively. It also aims to be Go-like: events
are transmitted via channels, and the library is fully safe for concurrent use.

To use the library, place your main program code in another function and call
Go(), passing that function as a parameter. (This is necessary due to threading
restrictions on some environments, such as Cocoa.) Once in the function you pass
to Go(), you can safely use the rest of the library. When this function returns,
so does Go(), and package functions become unavailable.

Building GUIs is as simple as creating a Window, populating it with Controls,
and then calling Open() on the Window. A Window only has one Control: you pack
multiple Controls into a Window by arranging them in layouts (Layouts are also
Controls). There are presently two Layouts, Stack and Grid, each with different
semantics on sizing and placement. See their documentation.

Once a Window is open, you cannot make layout or event channel changes.

Once your Window is open, you can begin to handle events. Handling events is
simple: because all events are channels exposed as exported members of the
Window and Control types, simply select on them. Event channels are initialized
by default. However, before you Open a Window, you can freely reassign event
channels, such that multiple events trigger the same channel, making event logic
more compact. You may also choose not to handle events; events are sent
asynchronously so the GUI loop is not initerrupted.

Here is a simple, complete program that asks the user for their name and greets
them after clicking a button.

    package main

    import (
    	"github.com/andlabs/ui"
    )

    func myMain() {
    	w := ui.NewWindow("Hello", 400, 100)
    	ui.AppQuit = w.Closing		// treat quitting the application like closing the main window
    	nameField := ui.NewLineEdit("Enter Your Name Here")
    	button := ui.NewButton("Click Here For a Greeting")
    	w.Open(ui.NewVerticalStack(nameField, button))

    	for {
    		select {
    		case <-w.Closing:		// user tries to close the window or quit the program
    			return
    		case <-button.Clicked:	// user clicked the button
    			ui.MsgBox(w, "Hello, " + nameField.Text() + "!", "")
    		}
    	}
    }

    func main() {
    	err := ui.Go(myMain)
    	if err != nil {
    		panic(err)
    	}
    }


### On Dialogs

The following functions provide dialogs. They exist both in package scope and as
methods on Window.

    MsgBox()
    MsgBoxError()

Dialogs opened by using the package-scope functions are modal to the entire
application: the user cannot interact with any other window until they are
dismissed. Whether or not resizing Windows will still be allowed is
implementation-defined; if the implementation does allow it, resizes will still
work properly. Whether or not the dialog box stays above all other Windows in
the program is also implementation-defined.

Dialogs opened by using the Window methods are modal to the receiver Window
only. Attempts to interact with the receiver Window will be blocked, but all
other Windows in the application can still be used properly. The dialog box will
also stay above the receiver Window. Whether the receiver Window can be resized
while the dialog box is displayed is implementation-defined, but will work
properly if allowed. If the receiver Window has not yet been created, the
methods panic. If the receiver Window has not been shown yet or is currently
hidden, what the methods do is implementation-defined.

The return type also differs between the two types of functions. Both ultimately
either yield a signal that the dialog has been dismissed or a code specifying
what the user decided to do with the dialog (if it has multiple choices). The
package-scope functions wait for the dialog box to be dismissed and merely
return the code (or nothing if no code is needed). The Window methods return
immediately with a channel that will eventually receive either the signal or the
return code. Package ui does not close these channels, nor does it send multiple
values on the same channel.


### Scrollbars

The following Controls have scrolling support built in:

    Listbox *
    Area

All of the above controls have both horizontal and vertical scrollbars. These
scrollbars hide themselves when not needed.

[FUTURE DISCUSSIONS: scrolling programmatically, MouseEvent/KeyEvent scroll
overrides]

[FUTURE PLAN: Controls that are not marked with a * in the above list can have
their scrollbars disabled completely in code.]

The result of resizing the window such that the scrollbars consider themselves
too small is implementation-defined.

autogenerated by windowsconstgen; do not edit

autogenerated by windowsconstgen; do not edit

## Usage

```go
var AppQuit chan struct{}
```
AppQuit is pulsed when the user decides to quit the program if their operating
system provides a facility for quitting an entire application, rather than
merely close all windows (for instance, Mac OS X via the Dock icon). You should
assign one of your Windows's Closing to this variable so the user choosing to
quit the application is treated the same as closing that window. If you do not
respond to this signal, nothing will happen; regardless of whether or not you
respond to this signal, the application will not quit. Do not merely check this
channel alone; it is not guaranteed to be pulsed on all systems or in all
conditions.

#### func  Go

```go
func Go(main func()) error
```
Go sets up the UI environment and runs main in a goroutine. If initialization
fails, Go returns an error and main is not called. Otherwise, Go does not return
to its caller until main does, at which point it returns nil. After it returns,
you cannot call future ui functions/methods meaningfully.

It is not safe to call ui.Go() in a goroutine. It must be called directly from
main().

This model is undesirable, but Cocoa limitations require it.

Go does not process the command line for flags (that is, it does not call
flag.Parse()), nor does package ui add any of the underlying toolkit's supported
command-line flags. If you must, and if the toolkit also has environment
variable equivalents to these flags (for instance, GTK+), use those instead.

#### func  MsgBox

```go
func MsgBox(primaryText string, secondaryText string)
```
MsgBox displays an informational message box to the user with just an OK button.
primaryText should be a short string describing the message, and will be
displayed with additional emphasis on platforms that support it. Optionally,
secondaryText can be used to show additional information. If you pass an empty
string for secondaryText, neither additional information nor space for
additional information will be shown. On platforms that allow for the message
box window to have a title, os.Args[0] is used.

See "On Dialogs" in the package overview for behavioral information.

#### func  MsgBoxError

```go
func MsgBoxError(primaryText string, secondaryText string)
```
MsgBoxError displays a message box to the user with just an OK button and an
icon indicating an error. Otherwise, it behaves like MsgBox.

See "On Dialogs" in the package overview for more information.

#### type Area

```go
type Area struct {
}
```

Area represents a blank canvas upon which programs may draw anything and receive
arbitrary events from the user. An Area has an explicit size, represented in
pixels, that may be different from the size shown in its Window. For information
on scrollbars, see "Scrollbars" in the Overview. The coordinate system of an
Area always has an origin of (0,0) which maps to the top-left corner; all
image.Points and image.Rectangles sent across Area's channels conform to this.
The size of an Area must be at least 1x1 (that is, neither its width nor its
height may be zero or negative). For control layout purposes, an Area prefers to
be at the size you set it to (so if an Area is not stretchy in its layout, it
will ask to have that size).

To handle events to the Area, an Area must be paired with an AreaHandler. See
AreaHandler for details.

Do not use an Area if you intend to read text. Area reads keys based on their
position on a standard 101-key keyboard, and does no character processing.
Character processing methods differ across operating systems; trying ot recreate
these yourself is only going to lead to trouble. [FOR FUTURE PLANNING Use
TextArea instead, providing a TextAreaHandler.]

#### func  NewArea

```go
func NewArea(width int, height int, handler AreaHandler) *Area
```
NewArea creates a new Area with the given size and handler. It panics if handler
is nil or if width or height is zero or negative.

#### func (*Area) RepaintAll

```go
func (a *Area) RepaintAll()
```
RepaintAll signals the entirety of the Area for redraw. If called before the
Window containing the Area is created, RepaintAll does nothing.

#### func (*Area) SetSize

```go
func (a *Area) SetSize(width int, height int)
```
SetSize sets the Area's internal drawing size. It has no effect on the actual
control size. SetSize is safe for concurrent use; if the Area is being repainted
or is handling an event, SetSize will wait for that to complete before changing
the Area's size. SetSize will also signal the entirety of the Area to be redrawn
as in RepaintAll. It panics if width or height is zero or negative.

#### type AreaHandler

```go
type AreaHandler interface {
	// Paint is called when the Area needs to be redrawn.
	// The part of the Area that needs to be redrawn is stored in cliprect.
	// Before Paint() is called, this region is cleared with a system-defined background color.
	// You MUST handle this event, and you MUST return a valid image, otherwise deadlocks and panicking will occur.
	// The image returned must have the same size as rect (but does not have to have the same origin points).
	// Example:
	// 	imgFromFile, _, err := image.Decode(file)
	// 	if err != nil { panic(err) }
	// 	img := image.NewRGBA(imgFromFile.Rect)
	// 	draw.Draw(img, img.Rect, imgFromFile, image.ZP, draw.Over)
	// 	// ...
	// 	func (h *myAreaHandler) Paint(rect image.Rectangle) *image.RGBA {
	// 		return img.SubImage(rect).(*image.RGBA)
	// 	}
	Paint(cliprect image.Rectangle) *image.RGBA

	// Mouse is called when the Area receives a mouse event.
	// You are allowed to do nothing in this handler (to ignore mouse events).
	// See MouseEvent for details.
	// If repaint is true, the Area is marked as needing to be redrawn.
	// After handling the mouse event, package ui will decide whether to perform platform-dependent event chain continuation based on that platform's designated action (so it is not possible to override global mouse events this way).
	Mouse(e MouseEvent) (repaint bool)

	// Key is called when the Area receives a keyboard event.
	// You are allowed to do nothing in this handler (to ignore keyboard events).
	// See KeyEvent for details.
	// If repaint is true, the Area is marked as needing to be redrawn.
	// After handling the key event, package ui will decide whether to perform platform-dependent event chain continuation based on that platform's designated action (so it is not possible to override global key events, such as Alt-Tab, this way).
	Key(e KeyEvent) (repaint bool)
}
```

AreaHandler represents the events that an Area should respond to. These methods
are all executed on the main goroutine, not necessarily the same one that you
created the AreaHandler in; you are responsible for the thread safety of any
members of the actual type that implements ths interface. (Having to use this
interface does not strike me as being particularly Go-like, but the nature of
Paint makes channel-based event handling a non-option; in practice, deadlocks
occur.)

#### type Button

```go
type Button struct {
	// Clicked gets a message when the button is clicked.
	// You cannot change it once the Window containing the Button has been created.
	// If you do not respond to this signal, nothing will happen.
	Clicked chan struct{}
}
```

A Button represents a clickable button with some text.

#### func  NewButton

```go
func NewButton(text string) (b *Button)
```
NewButton creates a new button with the specified text.

#### func (*Button) SetText

```go
func (b *Button) SetText(text string)
```
SetText sets the button's text.

#### func (*Button) Text

```go
func (b *Button) Text() string
```
Text returns the button's text.

#### type Checkbox

```go
type Checkbox struct {
}
```

A Checkbox is a clickable square with a label. The square can be either checked
or unchecked. Checkboxes start out unchecked.

#### func  NewCheckbox

```go
func NewCheckbox(text string) (c *Checkbox)
```
NewCheckbox creates a new checkbox with the specified text.

#### func (*Checkbox) Checked

```go
func (c *Checkbox) Checked() bool
```
Checked() returns whether or not the checkbox has been checked.

#### func (*Checkbox) SetText

```go
func (c *Checkbox) SetText(text string)
```
SetText sets the checkbox's text.

#### func (*Checkbox) Text

```go
func (c *Checkbox) Text() string
```
Text returns the checkbox's text.

#### type Combobox

```go
type Combobox struct {
}
```

A Combobox is a drop-down list of items, of which at most one can be selected at
any given time. You may optionally make the combobox editable to allow custom
items. Initially, no item will be selected (and no text entered in an editable
Combobox's entry field). What happens to the text shown in a Combobox if its
width is too small is implementation-defined.

#### func  NewCombobox

```go
func NewCombobox(items ...string) *Combobox
```
NewCombobox makes a new Combobox with the given items.

#### func  NewEditableCombobox

```go
func NewEditableCombobox(items ...string) *Combobox
```
NewEditableCombobox makes a new editable Combobox with the given items.

#### func (*Combobox) Append

```go
func (c *Combobox) Append(what ...string)
```
Append adds items to the end of the Combobox's list. Append will panic if
something goes wrong on platforms that do not abort themselves.

#### func (*Combobox) Delete

```go
func (c *Combobox) Delete(index int)
```
Delete removes the given item from the Combobox. It panics if the given index is
out of bounds.

#### func (*Combobox) InsertBefore

```go
func (c *Combobox) InsertBefore(what string, before int)
```
InsertBefore inserts a new item in the Combobox before the item at the given
position. It panics if the given index is out of bounds. InsertBefore will also
panic if something goes wrong on platforms that do not abort themselves.

#### func (*Combobox) Len

```go
func (c *Combobox) Len() int
```
Len returns the number of items in the Combobox.

On platforms for which this function may return an error, it panics if one is
returned.

#### func (*Combobox) SelectedIndex

```go
func (c *Combobox) SelectedIndex() int
```
SelectedIndex returns the index of the current selection in the Combobox. It
returns -1 either if no selection was made or if text was manually entered in an
editable Combobox.

#### func (*Combobox) Selection

```go
func (c *Combobox) Selection() string
```
Selection returns the current selection.

#### type Control

```go
type Control interface {
	// contains filtered or unexported methods
}
```

A Control represents an UI control. Note that Control contains unexported
members; this has the consequence that you can't build custom controls that
interface directly with the system-specific code (fo rinstance, to import an
unsupported control), or at least not without some hackery. If you want to make
your own controls, create an Area and provide an AreaHandler that does what you
need.

#### func  Space

```go
func Space() Control
```
Space returns a null Control intended for padding layouts with blank space. It
appears to its owner as a Control of 0x0 size.

For a Stack, Space can be used to insert spaces in the beginning or middle of
Stacks (Stacks by nature handle spaces at the end themselves). In order for this
to work properly, make the Space stretchy.

For a Grid, Space can be used to have an empty cell. A stretchy Grid cell with a
Space can be used to anchor the perimeter of a Grid to the respective Window
edges without making one of the other controls stretchy instead (leaving empty
space in the Window otherwise). Otherwise, you do not need to do anything
special for the Space to work (though remember that an entire row or column of
Spaces will appear as having height or width zero, respectively, unless one is
marked as stretchy).

The value returned from Space() may or may not be unique.

#### type ExtKey

```go
type ExtKey uintptr
```

ExtKey represents keys that are not in the typewriter section of the keyboard.

```go
const (
	Escape ExtKey = iota + 1
	Insert        // equivalent to "Help" on Apple keyboards
	Delete
	Home
	End
	PageUp
	PageDown
	Up
	Down
	Left
	Right
	F1 // F1..F12 are guaranteed to be consecutive
	F2
	F3
	F4
	F5
	F6
	F7
	F8
	F9
	F10
	F11
	F12
	N0 // numpad keys; independent of Num Lock state
	N1 // N0..N9 are guaranteed to be consecutive
	N2
	N3
	N4
	N5
	N6
	N7
	N8
	N9
	NDot
	NEnter
	NAdd
	NSubtract
	NMultiply
	NDivide
)
```

#### type Grid

```go
type Grid struct {
}
```

A Grid arranges Controls in a two-dimensional grid. The height of each row and
the width of each column is the maximum preferred height and width
(respectively) of all the controls in that row or column (respectively).
Controls are aligned to the top left corner of each cell. All Controls in a Grid
maintain their preferred sizes by default; if a Control is marked as being
"filling", it will be sized to fill its cell. Even if a Control is marked as
filling, its preferred size is used to calculate cell sizes. One Control can be
marked as "stretchy": when the Window containing the Grid is resized, the cell
containing that Control resizes to take any remaining space; its row and column
are adjusted accordingly (so other filling controls in the same row and column
will fill to the new height and width, respectively). A stretchy Control
implicitly fills its cell. All cooridnates in a Grid are given in (row,column)
form with (0,0) being the top-left cell.

#### func  NewGrid

```go
func NewGrid(nPerRow int, controls ...Control) *Grid
```
NewGrid creates a new Grid with the given Controls. NewGrid needs to know the
number of Controls in a row (alternatively, the number of columns); it will
determine the number in a column from the number of Controls given. NewGrid
panics if not given a full grid of Controls. Example:

    grid := NewGrid(3,
    	control00, control01, control02,
    	control10, control11, control12,
    	control20, control21, control22)

#### func (*Grid) SetFilling

```go
func (g *Grid) SetFilling(row int, column int)
```
SetFilling marks the given Control of the Grid as filling its cell instead of
staying at its preferred size. This function cannot be called after the Window
that contains the Grid has been created. It panics if the given coordinate is
invalid.

#### func (*Grid) SetStretchy

```go
func (g *Grid) SetStretchy(row int, column int)
```
SetStretchy marks the given Control of the Grid as stretchy. Stretchy implies
filling. Only one control can be stretchy per Grid; calling SetStretchy multiple
times merely changes which control is stretchy. This function cannot be called
after the Window that contains the Grid has been created. It panics if the given
coordinate is invalid.

#### type KeyEvent

```go
type KeyEvent struct {
	// Key is a byte representing a character pressed
	// in the typewriter section of the keyboard.
	// The value, which is independent of whether the
	// Shift key is held, is a constant with one of the
	// following (case-sensitive) values, drawn according
	// to the key's position on the keyboard.
	//    ` 1 2 3 4 5 6 7 8 9 0 - =
	//     q w e r t y u i o p [ ] \
	//      a s d f g h j k l ; '
	//       z x c v b n m , . /
	// The actual key entered will be the key at the respective
	// position on the user's keyboard, regardless of the actual
	// layout. (Some keyboards move \ to either the row above
	// or the row below but in roughly the same spot; this is
	// accounted for. Some keyboards have an additonal key
	// to the left of 'z' or additional keys to the right of '='; these
	// cannot be read.)
	// In addition, Key will contain
	// - ' ' (space) if the spacebar was pressed
	// - '\t' if Tab was pressed, regardless of Modifiers
	// - '\n' if the typewriter Enter key was pressed
	// - '\b' if the typewriter Backspace key was pressed
	// If this value is zero, see ExtKey.
	Key byte

	// If Key is zero, ExtKey contains a predeclared identifier
	// naming an extended key. See ExtKey for details.
	// If both Key and ExtKey are zero, a Modifier by itself
	// was pressed. Key and ExtKey will not both be nonzero.
	ExtKey ExtKey

	// If both Key and ExtKey are zero, Modifier will contain exactly one of its bits set, indicating which Modifier was pressed or released.
	// As with Modifiers itself, there is no way to differentiate between left and right modifier keys.
	// As such, the result of pressing and/or releasing both left and right of the same Modifier is system-defined.
	// Furthermore, the result of holding down a Key or ExtKey, then pressing a Modifier, and then releasing the original key is system-defined.
	// Under no condition shall Key, ExtKey, AND Modifier all be zero.
	Modifier Modifiers

	// Modifiers contains all the modifier keys currently being held at the time of the KeyEvent.
	// If Modifier is nonzero, Modifiers will not contain Modifier itself.
	Modifiers Modifiers

	// If Up is true, the key was released; if not, the key was pressed.
	// There is no guarantee that all pressed keys shall have
	// corresponding release events (for instance, if the user switches
	// programs while holding the key down, then releases the key).
	// Keys that have been held down are reported as multiple
	// key press events.
	Up bool
}
```

A KeyEvent represents a keypress in an Area.

Key presses are based on their positions on a standard 101-key keyboard found on
most computers. The names chosen for keys here are based on their names on US
English QWERTY keyboards; see Key for details.

If a key is pressed that is not supported by Key, ExtKey, or Modifiers, no
KeyEvent will be produced.

#### func (KeyEvent) EffectiveKey

```go
func (e KeyEvent) EffectiveKey() byte
```
EffectiveKey returns e.Key if it is set. Otherwise, if e.ExtKey denotes a numpad
key, EffectiveKey returns the equivalent e.Key value ('0'..'9', '.', '\n', '+',
'-', '*', or '/'). Otherwise, EffectiveKey returns zero.

#### type Label

```go
type Label struct {
}
```

A Label is a static line of text used to mark other controls. Label text is
drawn on a single line; text that does not fit is truncated. TODO vertical
alignment

#### func  NewLabel

```go
func NewLabel(text string) *Label
```
NewLabel creates a new Label with the specified text.

#### func (*Label) SetText

```go
func (l *Label) SetText(text string)
```
SetText sets the Label's text.

#### func (*Label) Text

```go
func (l *Label) Text() string
```
Text returns the Label's text.

#### type LineEdit

```go
type LineEdit struct {
}
```

A LineEdit is a control which allows you to enter a single line of text.

#### func  NewLineEdit

```go
func NewLineEdit(text string) *LineEdit
```
NewLineEdit makes a new LineEdit with the specified text.

#### func  NewPasswordEdit

```go
func NewPasswordEdit() *LineEdit
```
NewPasswordEdit makes a new LineEdit which allows the user to enter a password.

#### func (*LineEdit) SetText

```go
func (l *LineEdit) SetText(text string)
```
SetText sets the LineEdit's text.

#### func (*LineEdit) Text

```go
func (l *LineEdit) Text() string
```
Text returns the LineEdit's text.

#### type Listbox

```go
type Listbox struct {
}
```

A Listbox is a vertical list of items, of which either at most one or any number
of items can be selected at any given time. On creation, no item is selected.
For information on scrollbars, see "Scrollbars" in the Overview. Due to
implementation issues, the presence of horizontal scrollbars is currently
implementation-defined.

#### func  NewListbox

```go
func NewListbox(items ...string) *Listbox
```
NewListbox creates a new single-selection Listbox with the given items loaded
initially.

#### func  NewMultiSelListbox

```go
func NewMultiSelListbox(items ...string) *Listbox
```
NewMultiSelListbox creates a new multiple-selection Listbox with the given items
loaded initially.

#### func (*Listbox) Append

```go
func (l *Listbox) Append(what ...string)
```
Append adds items to the end of the Listbox's list. Append will panic if
something goes wrong on platforms that do not abort themselves.

#### func (*Listbox) Delete

```go
func (l *Listbox) Delete(index int)
```
Delete removes the given item from the Listbox. It panics if the given index is
out of bounds.

#### func (*Listbox) InsertBefore

```go
func (l *Listbox) InsertBefore(what string, before int)
```
InsertBefore inserts a new item in the Listbox before the item at the given
position. It panics if the given index is out of bounds. InsertBefore will also
panic if something goes wrong on platforms that do not abort themselves.

#### func (*Listbox) Len

```go
func (l *Listbox) Len() int
```
Len returns the number of items in the Listbox.

On platforms for which this function may return an error, it panics if one is
returned.

#### func (*Listbox) SelectedIndices

```go
func (l *Listbox) SelectedIndices() []int
```
SelectedIndices returns a list of the currently selected indexes in the Listbox,
or an empty list if none have been selected. This list will have at most one
item on a single-selection Listbox.

#### func (*Listbox) Selection

```go
func (l *Listbox) Selection() []string
```
Selection returns a list of strings currently selected in the Listbox, or an
empty list if none have been selected. This list will have at most one item on a
single-selection Listbox.

#### type Modifiers

```go
type Modifiers uintptr
```

Modifiers indicates modifier keys being held during an event. There is no way to
differentiate between left and right modifier keys. As such, what KeyEvents get
sent if the user does something unusual with both of a certain modifier key at
once is undefined.

```go
const (
	Ctrl  Modifiers = 1 << iota // the keys labelled Ctrl or Control on all platforms
	Alt                         // the keys labelled Alt or Option or Meta on all platforms
	Shift                       // the Shift keys
	Super                       // the Super keys on platforms that have one, or the Windows keys on Windows, or the Command keys on Mac OS X
)
```

#### type MouseEvent

```go
type MouseEvent struct {
	// Pos is the position of the mouse in the Area at the time of the event.
	Pos image.Point

	// If the event was generated by a mouse button being pressed, Down contains the ID of that button.
	// Otherwise, Down contains 0.
	// If Down contains nonzero, the Area will also receive keyboard focus.
	Down uint

	// If the event was generated by a mouse button being released, Up contains the ID of that button.
	// Otherwise, Up contains 0.
	// If both Down and Up are 0, the event represents mouse movement (with optional held buttons for dragging; see below).
	// Down and Up shall not both be nonzero.
	Up uint

	// If Down is nonzero, Count indicates the number of clicks: 1 for single-click, 2 for double-click, 3 for triple-click, and so on.
	// The order of events will be Down:Count=1 -> Up -> Down:Count=2 -> Up -> Down:Count=3 -> Up -> ...
	Count uint

	// Modifiers is a bit mask indicating the modifier keys being held during the event.
	Modifiers Modifiers

	// Held is a slice of button IDs that indicate which mouse buttons are being held during the event.
	// Held will not include Down and Up.
	// Held will be sorted.
	// Only buttons 1, 2, and 3 are guaranteed to be detected by Held properly; whether or not any others are is implementation-defined.
	//
	// If Held is non-empty but Up and Down are both zero, the mouse is being dragged, with all the buttons in Held being held.
	// Whether or not a drag into an Area generates MouseEvents is implementation-defined.
	// Whether or not a drag over an Area when the program is inactive generates MouseEvents is also implementation-defined.
	// Moving the mouse over an Area when the program is inactive and no buttons are held will, however, generate MouseEvents.
	Held []uint
}
```

MouseEvent contains all the information for a mous event sent by Area.Mouse.
Mouse button IDs start at 1, with 1 being the left mouse button, 2 being the
middle mouse button, and 3 being the right mouse button. If additional buttons
are supported, they will be returned with 4 being the first additional button.
For example, on Unix systems where mouse buttons 4 through 7 are pseudobuttons
for the scroll wheel directions, the next button, button 8, will be returned as
4, 9 as 5, etc. The association between button numbers and physical buttons are
system-defined. For example, on Windows, buttons 4 and 5 are mapped to what are
internally referred to as "XBUTTON1" and "XBUTTON2", which often correspond to
the dedicated back/forward navigation buttons on the sides of many mice. The
examples here are NOT a guarantee as to how many buttons maximum will be
available on a given system.

If the user clicked on the Area to switch to the Window it is contained in from
another window in the OS, the Area will receive a MouseEvent for that click.

#### func (MouseEvent) HeldBits

```go
func (e MouseEvent) HeldBits() (h uintptr)
```
HeldBits returns Held as a bit mask. Bit 0 maps to button 1, bit 1 maps to
button 2, etc.

#### type ProgressBar

```go
type ProgressBar struct {
}
```

A ProgressBar is a horizontal rectangle that fills up from left to right to
indicate the progress of a long-running task. This progress is represented by an
integer within the range [0,100], representing a percentage. Alternatively, a
progressbar can show an animation indicating that progress is being made but how
much is indeterminate. Newly-created ProgressBars default to showing 0%
progress.

#### func  NewProgressBar

```go
func NewProgressBar() *ProgressBar
```
NewProgressBar creates a new ProgressBar.

#### func (*ProgressBar) SetProgress

```go
func (p *ProgressBar) SetProgress(percent int)
```
SetProgress sets the currently indicated progress amount on the ProgressBar. If
percent is in the range [0,100], the progressBar shows that much percent
complete. If percent is -1, the ProgressBar is made indeterminate. Otherwise,
SetProgress panics. Calling SetProgress(-1) repeatedly will neither leave
indeterminate mode nor stop any animation involved in indeterminate mode
indefinitely; any other side-effect of doing so is implementation-defined.

#### type Stack

```go
type Stack struct {
}
```

A Stack stacks controls horizontally or vertically within the Stack's parent. A
horizontal Stack gives all controls the same height and their preferred widths.
A vertical Stack gives all controls the same width and their preferred heights.
Any extra space at the end of a Stack is left blank. Some controls may be marked
as "stretchy": when the Window they are in changes size, stretchy controls
resize to take up the remaining space after non-stretchy controls are laid out.
If multiple controls are marked stretchy, they are alloted equal distribution of
the remaining space.

#### func  NewHorizontalStack

```go
func NewHorizontalStack(controls ...Control) *Stack
```
NewHorizontalStack creates a new Stack that arranges the given Controls
horizontally.

#### func  NewVerticalStack

```go
func NewVerticalStack(controls ...Control) *Stack
```
NewVerticalStack creates a new Stack that arranges the given Controls
vertically.

#### func (*Stack) SetStretchy

```go
func (s *Stack) SetStretchy(index int)
```
SetStretchy marks a control in a Stack as stretchy. This cannot be called once
the Window containing the Stack has been created. It panics if index is out of
range.

#### type Window

```go
type Window struct {
	// Closing gets a message when the user clicks the window's close button.
	// You cannot change it once the Window has been created.
	// If you do not respond to this signal, nothing will happen; regardless of whether you handle the signal or not, the window will not be closed.
	Closing chan struct{}
}
```

Window represents an on-screen window.

#### func  NewWindow

```go
func NewWindow(title string, width int, height int) *Window
```
NewWindow allocates a new Window with the given title and size. The window is
not created until a call to Create() or Open().

#### func (*Window) Create

```go
func (w *Window) Create(control Control)
```
Create creates the Window, setting its control to the given control. It does not
show the window. This can only be called once per window, and finalizes all
initialization of the control.

#### func (*Window) Hide

```go
func (w *Window) Hide()
```
Hide hides the window.

#### func (*Window) MsgBox

```go
func (w *Window) MsgBox(primaryText string, secondaryText string) (done chan struct{})
```
MsgBox is the Window method version of the package-scope function MsgBox. See
that function's documentation and "On Dialogs" in the package overview for more
information.

#### func (*Window) MsgBoxError

```go
func (w *Window) MsgBoxError(primaryText string, secondaryText string) (done chan struct{})
```
MsgBoxError is the Window method version of the package-scope function
MsgBoxError. See that function's documentation and "On Dialogs" in the package
overview for more information.

#### func (*Window) Open

```go
func (w *Window) Open(control Control)
```
Open creates the Window with Create and then shows the Window with Show. As with
Create, you cannot call Open more than once per window.

#### func (*Window) SetSize

```go
func (w *Window) SetSize(width int, height int) (err error)
```
SetSize sets the window's size.

#### func (*Window) SetTitle

```go
func (w *Window) SetTitle(title string)
```
SetTitle sets the window's title.

#### func (*Window) Show

```go
func (w *Window) Show()
```
Show shows the window.