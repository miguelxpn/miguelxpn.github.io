---
layout: post
title:  "Hack The Boo 2023 - Writeups"
date:   2023-10-28 22:36:25 -0300
categories: security
---

## Hack The Boo

Hack The Boo 2023 was a CTF event held by [hackthebox][hackthebox], I never really participated in any public CTF events but always wanted to try it.

In this post I'll explain how I solved a few interesting challenges.

[hackthebox]: https://ctf.hackthebox.com/

## Spellbrewery

This challenge was a reverse engineering one. It gave us a zip file with a Dotnet app and a DLL file. When executing the app you'd get a prompt like this:

```
1. List Ingredients
2. Display Current Recipe
3. Add Ingredient
4. Brew Spell
5. Clear Recipe
6. Quit
```

The objective is to brew a spell with the correct ingredients, that should give us the flag. By brewing the spell with the wrong ingredients we get a error message and nothing else.

After some googling I found out that [ILSpy][ILSpy] might be a good tool to decompile Dotnet apps. After setting it up I managed to decompile the DLL. Inside it I found the implementation of the `brewSpell` method:

```c#
private static void BrewSpell()
{
    if (recipe.get_Count() < 1)
    {
        Console.WriteLine("You can't brew with an empty cauldron");
        return;
    }
    byte[] array = Enumerable.ToArray<byte>(Enumerable.Select<Ingredient, byte>((System.Collections.Generic.IEnumerable<Ingredient>)recipe, (Func<Ingredient, byte>)((Ingredient ing) => (byte)(System.Array.IndexOf<string>(IngredientNames, ((object)ing).ToString()) + 32))));
    if (Enumerable.SequenceEqual<Ingredient>((System.Collections.Generic.IEnumerable<Ingredient>)recipe, Enumerable.Select<string, Ingredient>((System.Collections.Generic.IEnumerable<string>)correct, (Func<string, Ingredient>)((string name) => new Ingredient(name)))))
    {
        Console.WriteLine("The spell is complete - your flag is: " + Encoding.get_ASCII().GetString(array));
        Environment.Exit(0);
    }
    else
    {
        Console.WriteLine("The cauldron bubbles as your ingredients melt away. Try another recipe.");
    }
}
```

The `correct` variable there looks suspicious, turns out it's a constant that we can inspect in the decompiled DLL:

```c#
private static readonly string[] correct = new string[36]
{
    "Phantom Firefly Wing", "Ghastly Gourd", "Hocus Pocus Powder", "Spider Sling Silk", "Goblin's Gold", "Wraith's Tear", "Werewolf Whisker", "Ghoulish Goblet", "Cursed Skull", "Dragon's Scale Shimmer",
    "Raven Feather", "Dragon's Scale Shimmer", "Zombie Zest Zest", "Ghoulish Goblet", "Werewolf Whisker", "Cursed Skull", "Dragon's Scale Shimmer", "Haunted Hay Bale", "Wraith's Tear", "Zombie Zest Zest",
    "Serpent Scale", "Wraith's Tear", "Cursed Crypt Key", "Dragon's Scale Shimmer", "Salamander's Tail", "Raven Feather", "Wolfsbane", "Frankenstein's Lab Liquid", "Zombie Zest Zest", "Cursed Skull",
    "Ghoulish Goblet", "Dragon's Scale Shimmer", "Cursed Crypt Key", "Wraith's Tear", "Black Cat's Meow", "Wraith Whisper"
};
```

After inputing that as the answer I got the flag!

[ILspy]: https://github.com/icsharpcode/ILSpy

## Spookycheck

This one was also a reverse engineering challenge. It gave us a `check.pyc` file. In the Python world .pyc files are basically files with the compiled python bytecode, we can execute them as if they were a regular python script.

By running it we'd get the following prompt:

```
🎃 Welcome to SpookyCheck 🎃
🎃 Enter your password for spooky evaluation 🎃
```

After some research I found out that there's a module included with Python that can be used for disassembly. I wrote the following short script to spit out the disassembled bytecode:

```python
import dis
import marshal

with open('check.pyc', 'rb') as f:
    f.seek(16)
    dis.dis(marshal.load(f))
```

Here's the full disassembled code:

```
  0           0 RESUME                   0

  1           2 LOAD_CONST               0 (b'SUP3RS3CR3TK3Y')
              4 STORE_NAME               0 (KEY)

  2           6 PUSH_NULL
              8 LOAD_NAME                1 (bytearray)
             10 LOAD_CONST               1 (b'\xe9\xef\xc0V\x8d\x8a\x05\xbe\x8ek\xd9yX\x8b\x89\xd3\x8c\xfa\xdexu\xbe\xdf1\xde\xb6\\')
             12 PRECALL                  1
             16 CALL                     1
             26 STORE_NAME               2 (CHECK)

  4          28 LOAD_CONST               2 (<code object transform at 0x1004ea1f0, file "check.py", line 4>)
             30 MAKE_FUNCTION            0
             32 STORE_NAME               3 (transform)

 10          34 LOAD_CONST               3 (<code object check at 0x1004eaa60, file "check.py", line 10>)
             36 MAKE_FUNCTION            0
             38 STORE_NAME               4 (check)

 13          40 LOAD_NAME                5 (__name__)
             42 LOAD_CONST               4 ('__main__')
             44 COMPARE_OP               2 (==)
             50 POP_JUMP_FORWARD_IF_FALSE    88 (to 228)

 14          52 PUSH_NULL
             54 LOAD_NAME                6 (print)
             56 LOAD_CONST               5 ('🎃 Welcome to SpookyCheck 🎃')
             58 PRECALL                  1
             62 CALL                     1
             72 POP_TOP

 15          74 PUSH_NULL
             76 LOAD_NAME                6 (print)
             78 LOAD_CONST               6 ('🎃 Enter your password for spooky evaluation 🎃')
             80 PRECALL                  1
             84 CALL                     1
             94 POP_TOP

 16          96 PUSH_NULL
             98 LOAD_NAME                7 (input)
            100 LOAD_CONST               7 ('👻 ')
            102 PRECALL                  1
            106 CALL                     1
            116 STORE_NAME               8 (inp)

 17         118 PUSH_NULL
            120 LOAD_NAME                4 (check)
            122 LOAD_NAME                8 (inp)
            124 LOAD_METHOD              9 (encode)
            146 PRECALL                  0
            150 CALL                     0
            160 PRECALL                  1
            164 CALL                     1
            174 POP_JUMP_FORWARD_IF_FALSE    13 (to 202)

 18         176 PUSH_NULL
            178 LOAD_NAME                6 (print)
            180 LOAD_CONST               8 ("🦇 Well done, you're spookier than most! 🦇")
            182 PRECALL                  1
            186 CALL                     1
            196 POP_TOP
            198 LOAD_CONST              10 (None)
            200 RETURN_VALUE

 20     >>  202 PUSH_NULL
            204 LOAD_NAME                6 (print)
            206 LOAD_CONST               9 ('💀 Not spooky enough, please try again later 💀')
            208 PRECALL                  1
            212 CALL                     1
            222 POP_TOP
            224 LOAD_CONST              10 (None)
            226 RETURN_VALUE

 13     >>  228 LOAD_CONST              10 (None)
            230 RETURN_VALUE

Disassembly of <code object transform at 0x1004ea1f0, file "check.py", line 4>:
  4           0 RESUME                   0

  5           2 LOAD_CONST               1 (<code object <listcomp> at 0x100451f10, file "check.py", line 5>)
              4 MAKE_FUNCTION            0

  7           6 LOAD_GLOBAL              1 (NULL + enumerate)
             18 LOAD_FAST                0 (flag)
             20 PRECALL                  1
             24 CALL                     1

  5          34 GET_ITER
             36 PRECALL                  0
             40 CALL                     0
             50 RETURN_VALUE

Disassembly of <code object <listcomp> at 0x100451f10, file "check.py", line 5>:
  5           0 RESUME                   0
              2 BUILD_LIST               0
              4 LOAD_FAST                0 (.0)
        >>    6 FOR_ITER                54 (to 116)

  7           8 UNPACK_SEQUENCE          2
             12 STORE_FAST               1 (i)
             14 STORE_FAST               2 (f)

  6          16 LOAD_FAST                2 (f)
             18 LOAD_CONST               0 (24)
             20 BINARY_OP                0 (+)
             24 LOAD_CONST               1 (255)
             26 BINARY_OP                1 (&)
             30 LOAD_GLOBAL              0 (KEY)
             42 LOAD_FAST                1 (i)
             44 LOAD_GLOBAL              3 (NULL + len)
             56 LOAD_GLOBAL              0 (KEY)
             68 PRECALL                  1
             72 CALL                     1
             82 BINARY_OP                6 (%)
             86 BINARY_SUBSCR
             96 BINARY_OP               12 (^)
            100 LOAD_CONST               2 (74)
            102 BINARY_OP               10 (-)
            106 LOAD_CONST               1 (255)
            108 BINARY_OP                1 (&)

  5         112 LIST_APPEND              2
            114 JUMP_BACKWARD           55 (to 6)
        >>  116 RETURN_VALUE

Disassembly of <code object check at 0x1004eaa60, file "check.py", line 10>:
 10           0 RESUME                   0

 11           2 LOAD_GLOBAL              1 (NULL + transform)
             14 LOAD_FAST                0 (flag)
             16 PRECALL                  1
             20 CALL                     1
             30 LOAD_GLOBAL              2 (CHECK)
             42 COMPARE_OP               2 (==)
             48 RETURN_VALUE
```

A few pieces stand out, first we have those two constants called `KEY` and `CHECK`.

There's also a function called `transform`, with a parameter called flag that uses a list comprehension inside it, the list comprehension, seems like it's iterating over the flag variable and transforming it by doing a bunch of operations with some using the `KEY` constant. After all that there's a `check` function that calls transform over the `flag` variable and then checks it against the `CHECK` constant.

That seems to indicate that the `CHECK` constant has the encrypted flag, and it is encrypted using the `KEY` constant. So to get the flag we probably have to reverse the process. Here I decided to cheat a bit, I never laid eyes on Python bytecode before, but it all seemed rather structured and relatively higher leveled, so I wondered, if I give ChatGPT the bytecode for the transform function and the list comprehension, could it write me a script to reverse the process? Turns out it could, and here's what it wrote:

```python
KEY = b'SUP3RS3CR3TK3Y'
CHECK = b'\xe9\xef\xc0V\x8d\x8a\x05\xbe\x8ek\xd9yX\x8b\x89\xd3\x8c\xfa\xdexu\xbe\xdf1\xde\xb6\\'

def reverse_transform(byte_val, index):
    # Add 74
    byte_val += 74
    # XOR with the byte from KEY
    byte_val ^= KEY[(index + len(KEY)) % len(KEY)]
    # Subtract 24
    byte_val -= 24
    # Ensure it remains a byte value
    byte_val &= 255
    return byte_val

original_input = bytearray()
for i, byte_val in enumerate(CHECK):
    original_input.append(reverse_transform(byte_val, i))

print(original_input.decode())
```

It worked on the first try and I got the flag! Thanks ChatGPT!

## Ghostly Templates

This was a web challenge. They gave us a Golang app that looks like this:

```go
package main

import (
	"encoding/json"
	"fmt"
	"html/template"
	"io"
	"net/http"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
)

const WEB_PORT = "1337"
const TEMPLATE_DIR = "./templates"

type LocationInfo struct {
	Status      string  `json:"status"`
	Country     string  `json:"country"`
	CountryCode string  `json:"countryCode"`
	Region      string  `json:"region"`
	RegionName  string  `json:"regionName"`
	City        string  `json:"city"`
	Zip         string  `json:"zip"`
	Lat         float64 `json:"lat"`
	Lon         float64 `json:"lon"`
	Timezone    string  `json:"timezone"`
	ISP         string  `json:"isp"`
	Org         string  `json:"org"`
	AS          string  `json:"as"`
	Query       string  `json:"query"`
}

type MachineInfo struct {
	Hostname      string
	OS            string
	KernelVersion string
	Memory        string
}

type RequestData struct {
	ClientIP     string
	ClientUA     string
	ServerInfo   MachineInfo
	ClientIpInfo LocationInfo `json:"location"`
}

func GetServerInfo(command string) string {
	out, err := exec.Command("sh", "-c", command).Output()
	if err != nil {
		return ""
	}
	return string(out)
}

func (p RequestData) GetLocationInfo(endpointURL string) (*LocationInfo, error) {
	resp, err := http.Get(endpointURL)
	if err != nil {
		return nil, err
	}

	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("HTTP request failed with status code: %d", resp.StatusCode)
	}

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return nil, err
	}

	var locationInfo LocationInfo
	if err := json.Unmarshal(body, &locationInfo); err != nil {
		return nil, err
	}

	return &locationInfo, nil
}

func (p RequestData) IsSubdirectory(basePath, path string) bool {
	rel, err := filepath.Rel(basePath, path)
	if err != nil {
		return false
	}
	return !strings.HasPrefix(rel, ".."+string(filepath.Separator))
}

func (p RequestData) OutFileContents(filePath string) string {
	data, err := os.ReadFile(filePath)
	if err != nil {
		return err.Error()
	}
	return string(data)
}

func readRemoteFile(url string) (string, error) {
	response, err := http.Get(url)
	if err != nil {
		return "", err
	}

	defer response.Body.Close()

	if response.StatusCode != http.StatusOK {
		return "", fmt.Errorf("HTTP request failed with status code: %d", response.StatusCode)
	}

	content, err := io.ReadAll(response.Body)
	if err != nil {
		return "", err
	}

	return string(content), nil
}

func getIndex(w http.ResponseWriter, r *http.Request) {
	http.Redirect(w, r, "/view?page=index.tpl", http.StatusMovedPermanently)
}

func getTpl(w http.ResponseWriter, r *http.Request) {
	var page string = r.URL.Query().Get("page")
	var remote string = r.URL.Query().Get("remote")

	if page == "" {
		http.Error(w, "Missing required parameters", http.StatusBadRequest)
		return
	}

	reqData := &RequestData{}

	userIPCookie, err := r.Cookie("user_ip")
	clientIP := ""

	if err == nil {
		clientIP = userIPCookie.Value
	} else {
		clientIP = strings.Split(r.RemoteAddr, ":")[0]
	}

	userAgent := r.Header.Get("User-Agent")

	locationInfo, err := reqData.GetLocationInfo("https://freeipapi.com/api/json/" + clientIP)

	if err != nil {
		http.Error(w, "Could not fetch IP location info", http.StatusInternalServerError)
		return
	}

	reqData.ClientIP = clientIP
	reqData.ClientUA = userAgent
	reqData.ClientIpInfo = *locationInfo
	reqData.ServerInfo.Hostname = GetServerInfo("hostname")
	reqData.ServerInfo.OS = GetServerInfo("cat /etc/os-release | grep PRETTY_NAME | cut -d '\"' -f 2")
	reqData.ServerInfo.KernelVersion = GetServerInfo("uname -r")
	reqData.ServerInfo.Memory = GetServerInfo("free -h | awk '/^Mem/{print $2}'")

	var tmplFile string

	if remote == "true" {
		tmplFile, err = readRemoteFile(page)

		if err != nil {
			http.Error(w, "Internal Server Error", http.StatusInternalServerError)
			return
		}
	} else {
		if !reqData.IsSubdirectory("./", TEMPLATE_DIR+"/"+page) {
			http.Error(w, "Internal Server Error", http.StatusInternalServerError)
			return
		}

		tmplFile = reqData.OutFileContents(TEMPLATE_DIR + "/" + page)
	}

	tmpl, err := template.New("page").Parse(tmplFile)
	if err != nil {
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}

	err = tmpl.Execute(w, reqData)
	if err != nil {
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}
}

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("/", getIndex)
	mux.HandleFunc("/view", getTpl)
	mux.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.Dir("static"))))

	fmt.Println("Server started at port " + WEB_PORT)
	http.ListenAndServe(":"+WEB_PORT, mux)
}
```

Just like the other web challenge the flag was in a file inside the root of the container.

Basically the app exposes three things: 
- The index route which just redirects to the view route. 
- The view route which populates the RequestData struct and renders a template using it as data.
- The static route which is used to serve static content to the app.

The interesting route here is the view one, it has a bunch of interesting stuff, first of all the template is user supplied data, you can either specify a local template that's already in the app, or give it a url so it can download a template to render.

The first thing that called my attention was the `IsSubdirectory` function, my first thought was that if I could bypass the traversal protections there we could simply get the app the render the flag.txt file as a template and we'd see the flag. I had no luck bypassing it though so I decided to change my strategy.

After reading a bit about golang html/templates I learned that if you send a struct to the template context inside the template you get access to all the functions for that struct. If you look at the `RequestData` functions you'll see that `OutFileContents` is one of them, this function will accept a single parameter containing a file path and will return the contents of the file, exactly what we need, how convenient!

To exploit it we simply need to host somewhere a template that looks like this:
```html
<html> {% assign special = '{{ .OutFileContents "/flag.txt" }}' %}
<div>
{{ special }}
</div>
</html>
```

I used pastebin to upload the template and using it's url I got the flag.