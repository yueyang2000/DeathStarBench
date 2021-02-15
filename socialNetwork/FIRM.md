## FIRM Deployment

install dependencies

```bash
sudo apt-get install libssl-dev
sudo apt-get install libz-dev
sudo apt-get install luarocks
sudo luarocks install luasocket
```

work load generation

```bash
./wrk -D exp -t 8 -c 100 -R 1600 -d 1h -L -s ./scripts/social-network/compose-post.lua http://10.99.196.255:8080/wrk2-api/post/compose
```



