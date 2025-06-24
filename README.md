
<img alt="wkv logo" src="icon-shadow.png" width=256 height=256>

# wkv
wkv - a rough approximation of what a key-value config should've looked like.

## What?
wkv is a key-value data storage format designed to be fast and simple. No more,
no less.

wkv stands for Whatever Key-Values.

## But... where is all the code?
LMAO, where do you think it is?

On a more serious note, you write it. The format is really simple and does not
have _any_ specification. And that is intentional: it is really easy to add a
whole bunch of features for the format to fit all the use cases. But... why?
Isn't it easier for you to actually go and implement all the features you need?
That is the philosophy behind wkv. The philosophy is such that there isn't a
philosophy. You do what you want, based, pretty roughly, on what is written
here. It can be as simple or as complex as you want, ranging from a single C
loop with a few branch cases to a whole Python module. You need datetime?
Go ahead, implement that. You need HEX colors? Go ahead, it's that simple.

## Base case
At the _very_ base case wkv looks like this:
```
key=Value
Key is any sequence of bytes=Value is too.
```
And that is it. Anything more complex, and you are on your own. Do you want
escaped strings? Implement them:
```
key="Here's a newline -> \n"
```
Need a binary blob? Go ahead:
```
key=b26Here's another newline ->

another_one=sHere's an unescaped string
```
The concept is that everything is a single type - **data**. The assumptions you
make about it transform it into different types. Do you want to infer the types
from the definitions? Go ahead. You don't? Well, you don't. It's _that_ simple.

## This looks stupid. Don't we already have JSON, YAML, TOML, &lt;insert your favourite markup language here&gt;?

Yup, we certainly do. The problem is: they are all really complex. You need
whole libraries just to parse and serialize them. Yes, sometimes this complexity
is needed, e.g. for trees or something... But you most certainly do not need to
write a tree when you are trying to save a savestate of your application to the
disk. Most of the time you need a simple key-value storage. And that is it.

Well, what if you really want trees? Go ahead, implement them in wkv. And no one
will say anything about best practices and unadhering to the spec, because the
spec _does not exist_. You do whatever, as the name suggests.

## But what about &lt;a common data type&gt;?
You want booleans? Go ahead, prefix notation allows really easily to define them
as `t` and `f` (gives you those lisp vibes, yeah?). Null? `n`. A float number?
You could define it as a standard floating point literal from C, but that
requires a lot of machinery. What about `f1.2`? Or, maybe, implement all this
tricky machinery. The limit is your imagination.

I've already gone through the trouble of figuring most of these out, so, here's
a (kinda) comprehensive list of common data types:
- `[-]12`: a number, consisting only of digits. Optionally signed, with a minus in
  front of it.
- `s<data>\n`: a string, consisting of any data, unescaped, excluding the newline
- `"<string>"`: a quoted string, optionally escaped
- `b<length><exactly length bytes>`: a blob of opaque sequence of bytes
- `f12[.34][e56]`: a floating point literal
- `t`, `f`: booleans
- `n`: a null type

They are designed so it is real easy to parse them just by checking from what
charachter they start from, so your little loop does not have to go through the
long tropes of parsing the values.

## I want arrays!
Arrays are simple. They are merely a length + a stride of keys.

Example:

```
images.len=10
images.0=b1024<data>
images.1=b1024<data>
...
images.9=b1024<data>
```

Or maybe underscore instead of the dot? Or maybe a snowman emoji? Or maybe you
think it would be easier to parse if you added indices to it in a form of a uint
binary number? Or maybe store a length and an array as another key-value that
evaluates to an array for the compression of the file size? Again, your
creativity is your friend.

## Where do I even start?
Usually, wkv is implemented in a single loop with a few branching ifs.

Want examples?

```c
// Yes, this does use some abstract data structures. Technically, it is
// pseudocode, go cry about it, that's just an example. It should be relatively
// easy to implement this with only C's stdlib, but I also am a lazy person.
// That implementation is left as an exercise for the reader. ;)

/// Returns true on failure.
bool parse_config(FILE *f, struct Config *conf) {
    while (!feof(f) && !ferror(f)) {
        String value = file_read_until(f, "\n");
	String key = string_split(&value, SV("="));
	if (string_equal(key, SV("option1"))) conf->option1 = string_parse_int(value);
	else if (string_equal(key, SV("option2"))) conf->option2 = value;
	else return true;
    }
    return false;
}
```

```python
# A simple example
def parse_config(config):
    result = {}
    for i in config.split("\n"):
    	key, value = i.split("=")
        result[key] = value
    return result

# A more sophisticated one
# This one needs a lot more error checking, but it's probably okay for an
# example.
def parse_a_hard_config(config):
    result = {}
    cur = 0
    anchor = 0
    while cur < len(config):
    	while config[cur] != "=": cur += 1
	key = config[anchor:cur]
	cur += 1  # Skip the `=`
	value = None
	if config[cur] == 's':
	    anchor = cur + 1
	    while config[cur] != "\n": cur += 1
	    value = config[anchor:cur]
	elif config[cur] == 'b':
	    anchor = cur + 1
	    cur += 1
	    while config[cur] in string.digits: cur += 1
	    length = int(config[anchor:cur])
	    anchor = cur
	    cur += length
	    value = config[anchor:cur]
	elif config[cur] == 't': value = True
	elif config[cur] == 'f': value = False
	else: raise Exception('the parser shit the bed')
	result[key] = value
    return result
```

Yes, some of the examples lack error checking. But what if you parse only
machine-generated config that is always generated right? Well, you are certainly
lying to me, nothing the machine generates is right. But okay, suppose it
doesn't need as thorough of error checking as user input. Well, then don't
implement any of it! It's that simple.

What if you really want to change something in the format? Maybe you want to
swap the equals separator to the snowman emoji? Go ahead. This is not a format,
this is a rough approximation of a good config format. Or a savestate format. Or
a ... you get the point. If you really want you can even store an image as wkv,
maybe like this?
```
channels=4
width=1024
height=1024
data=b4194304{ raw image data }
```
As inefficient as this is, it is quite funny.

## This is really cool! What about licensing though?
Haha, what? **You** define the format in **your** code, write **your own**
parser... As far as I am concerned the thing you took from me is the mere idea,
and licensing concepts so far hasn't been a great idea. :^)

Anyway, feel free to use this README for anything. For making real
production-grade stuff, for hackernews praising for how simple this is and how
we do not need any more complexity and for hackernews raging for how simple this
is and how we need more complexity.

---

## r/programming chaos
Okay, I did not expect people to rage as much as they were over the fact that
you actually have to write code for this one. Like, who are you? _A programmer?_
No shit this isn't interoperable - I am not going to put this format into a
REST API, I am parsing some user data and some savestates with this. That is it. If this
makes your blood boil - I think I did a very well job here. Actually, the more people rage
\- the more I think I've honestly struct gold. For all the programmers reading this
\- have fun implmeneting this, for all the "programmers" - I do not want you to
get your diaper messy, go ahead and rage about this on some forums. &gt;:)

Seriously, the fact that REAL people that program FOR MONEY wrote this in the comments
honestly makes me want to get replaced by AI.
