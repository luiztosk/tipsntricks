
## Convert multiple ebooks from `.epub` to `.mobi` in parallel

Kindle accepts `.mobi` files directly, just copy them to the root `documents` folder. To convert from the more common `.epub`, we can use the `calibre` CLI utility `ebook-convert`.

To run multiple instances in parallel, converting all `.epub` files in current dir, and log each process into a `.log` with the same name, do:

```
foreach x (*.epub) {
    echo -e "\n$x";
    ebook-convert $x .mobi > "$x".log 2>&1 & 
}
```
In the 3rd line, `>` redirects `stdin` to the file `"$x".log`, and `2>&1` redirects stderr to stdin to log them together. The final `&` runs `ebook-convert` as a background process, that can be monitored in `htop` with the printed PID, or using the `jobs` utility.


## Connect to `ssh` host when it is online:
```
until nc -vzw 2 $host 22; do sleep 2; done && ssh $host
```
Very useful when rebooting hosts (VMs, IoT).