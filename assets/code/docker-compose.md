```
version: "2"
services:
  hello1:
    image: nginx
    container_name: hello1
    volumes:
      - "hello1:/stuff:ro"
    ports:
      - "80:80"

volumes:
  hello1:
    driver: "vsphere"
    driver_opts:
      Capacity: "1G"
      VolumeStore: "ds"

networks:
  default:
    external:
      name: public
```
