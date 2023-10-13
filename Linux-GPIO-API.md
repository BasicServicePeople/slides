---
type: slide
slideOptions:
  spotlight:
    enabled: true
---

<style>
.reveal {
  font-size: 24px;
}
</style>


# libgpiod
Zielgruppe: Applikations-Entwickler (auch HW) die in Linux GPIOs bedienen wollen.

---

### Was macht das OS eigentlich? 
 ---
![](https://hackmd.io/_uploads/BJUmfcUZ6.png)

---

### Userpace <-> Kernelspace am Beispiel GPIO-Driver
 ----

![](https://hackmd.io/_uploads/Bk7nRtUWT.png)


---

![](https://hackmd.io/_uploads/rkd5T38-6.png)

    
---

### sysfs = bad
 ---
- State not tied to process
- Concurrent access to sysfs attributes -> keine Zugriffskontrolle, kein locking
- If process crashes, the GPIOs remain exported (zwingt zu echo unexport usw.)
- Cumbersome API -> echo export beispiel
- Single sequence of GPIO numbers representing a two-level hierarchy - necessary to calculate the number of the GPIO

----

<!-- .slide: style="text-align: left" -->

GPIO Nummer ausrechnen:

GPIO-Nummer = ( GPIO-Bank-Nr * 16|32 ) + GPIO-Offset

z.b. GPIO #31 auf GPIO-Bank #4: 3*32+31=**127**

> Achtung: GPIO-Bank und GPIO-Offset zählen ab 0 (HWer und Manual zählen meist ab 1)

```bash=
echo 127 > /sys/class/gpio/export

echo out > /sys/class/gpio/gpio127/direction
echo 1 > /sys/class/gpio/gpio127/value

echo in > /sys/class/gpio/gpio127/direction
cat /sys/class/gpio/gpio127/value

echo 127 > /sys/class/gpio/unexport
```

---

###  GPIO character device - new user API
 ---
- v1.x Seit Linux v4.8 (2016)
- v2 seit Linux v5.10
- One device file per gpiochip (Bank)
  * `/dev/gpiochip0, /dev/gpiochip1, /dev/gpiochipX`…
- Similar to other kernel interfaces: open() + ioctl() + poll() + read() + close()
- Possible to request multiple lines at once (for reading/setting values)
- Possible to find GPIO lines and chips by name
- Open-source and open-drain flags, user/consumer strings, uevents
- Reliable polling

----

#### Racy GPIO event polling
 ---
- Z.B. Schalter an gpio und schnell drücken
-> Wettlaufsituation :wink: Wer ist schneller: Der Read-Code oder der GPIO-Impuls?

![](https://hackmd.io/_uploads/ByLuP2LZp.png)
sysfs: Events can get lost! | gpiod: Events never get lost!

----

#### Event polling mit sysfs
 ---
```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <poll.h>

#define GPIO_PIN "17"  // Change this to the GPIO pin you want to use

int main() {
    int gpio_fd;
    char gpio_path[256];
    struct pollfd pfd;

    // Export the GPIO pin
    gpio_fd = open("/sys/class/gpio/export", O_WRONLY);
    if (gpio_fd == -1) {
        perror("Unable to export GPIO");
        exit(1);
    }
    write(gpio_fd, GPIO_PIN, sizeof(GPIO_PIN));
    close(gpio_fd);

    // Set the direction of the GPIO pin to input
    sprintf(gpio_path, "/sys/class/gpio/gpio%s/direction", GPIO_PIN);
    gpio_fd = open(gpio_path, O_WRONLY);
    if (gpio_fd == -1) {
        perror("Unable to set direction");
        exit(1);
    }
    write(gpio_fd, "in", 2);
    close(gpio_fd);

    // Open the value file for reading
    sprintf(gpio_path, "/sys/class/gpio/gpio%s/value", GPIO_PIN);
    gpio_fd = open(gpio_path, O_RDONLY);
    if (gpio_fd == -1) {
        perror("Unable to open value file");
        exit(1);
    }

    pfd.fd = gpio_fd;
    pfd.events = POLLPRI | POLLERR;

    while (1) {
        int ret = poll(&pfd, 1, -1); // Wait for events indefinitely
        if (ret < 0) {
            perror("Poll error");
            exit(1);
        }
        if (pfd.revents & POLLPRI) {
            char buf[1];
            lseek(gpio_fd, 0, SEEK_SET); // Move to the beginning of the file
            read(gpio_fd, buf, 1);
            printf("GPIO value: %c\n", buf[0]);
        }
    }

    // Unexport the GPIO pin when done
    gpio_fd = open("/sys/class/gpio/unexport", O_WRONLY);
    if (gpio_fd == -1) {
        perror("Unable to unexport GPIO");
        exit(1);
    }
    write(gpio_fd, GPIO_PIN, sizeof(GPIO_PIN));
    close(gpio_fd);

    close(gpio_fd);

    return 0;
}
```

----

#### GPIO event polling - now reliable using libgpiod:
 ---
```c
#include <stdio.h>
#include <gpiod.h>

#define GPIO_CHIPNAME "gpiochip0"  // Modify this to match your GPIO chip
#define GPIO_LINE_OFFSET 17       // Modify this to match your GPIO pin number

int main() {
    struct gpiod_chip *chip;
    struct gpiod_line *line;

    // Initialize the GPIO chip
    chip = gpiod_chip_open_by_name(GPIO_CHIPNAME);
    if (!chip) {
        perror("Failed to open GPIO chip");
        return 1;
    }

    // Get the GPIO line
    line = gpiod_chip_get_line(chip, GPIO_LINE_OFFSET);
    if (!line) {
        perror("Failed to get GPIO line");
        gpiod_chip_close(chip);
        return 1;
    }

    // Request the line for input
    if (gpiod_line_request_input(line, "example-gpio") < 0) {
        perror("Failed to request GPIO line");
        gpiod_chip_close(chip);
        return 1;
    }

    while (1) {
        int value = gpiod_line_get_value(line);
        printf("GPIO value: %d\n", value);
    }

    // Release resources
    gpiod_line_release(line);
    gpiod_chip_close(chip);

    return 0;
}
```

---

### libgpiod

Userspace Library zur Interaktion mit der 

### Demo

- locking sysfs vs. gpiod