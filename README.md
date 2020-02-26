# sisop-modul-2

# Daemon dan Proses

Menggunakan:
* Linux
* Bahasa C (compile dengan _gcc_)

# Daftar Isi

* Daemon dan Proses
    * [Daftar Isi](#daftar-isi)
    * [Proses](#proses)
        * [1. Pengertian](#1-pengertian)
        * [2. Macam-Macam PID](#2-macam-macam-pid)
            * [2.1 User ID (UID)](#21-user-id-(uid))
            * [2.2 Process ID (PID)](#22-process-id-(pid))
            * [2.3 Parent PID (PPID)](#23-parent-pid-(ppid))
    * [Daemon](#daemon)
        * [1. Pengertian Daemon](#1-pengertian-daemon)
        * [2. Langkah Pembuatan Daemon](#2-langkah-pembuatan-daemon)
        * [3. Contoh Implementasi Daemon](#3-contoh-implementasi-daemon)

# Proses

## 1. Pengertian
Proses adalah kondisi dimana OS menjalankan (eksekusi) suatu program. Ketika suatu program tersebut dieksekusi oleh OS, proses tersebut memiliki PID (Process ID) yang merupakan identifier dari suatu proses. Pada UNIX, untuk melihat proses yang dieksekusi oleh OS dengan memanggil perintah shell ```ps```. Untuk melihat lebih lanjut mengenai perintah ```ps``` dapat membuka ```man ps```.

Dalam penggunaannya, suatu proses dapat membentuk proses lainnya yang disebut _spawning process_. Proses yang memanggil proses lainnya disebut _parent process_ dan yang terpanggil disebut _child process_.

## 2. Macam-Macam PID

### 2.1 User ID (UID)
Merupakan identifier dari suatu proses yang menampilkan user yang menjalankan suatu program. Pada program C, dapat memanggil fungsi ``` uid_t getuid(void);```

### 2.2 Process ID (PID)
Angka unik dari suatu proses yang sedang berjalan untuk mengidentifikasi suatu proses. Pada program C, dapat memanggil fungsi ```pid_t getpid(void);```

### 2.3 Parent PID (PPID)
Setiap proses memiliki identifier tersendiri dan juga setelah proses tersebut membuat proses lainnya. Proses yang terbentuk ini memiliki identifier berupa ID dari pembuatnya (parent). Pada program C, dapat memanggil fungsi ```
pid_t getppid(void);```.

## 3. Melihat Proses Berjalan
Untuk melihat proces yang sedang berjalan di OS, dapat menggunakan ```ps -ef``` untuk melihat secara detailnya.

![show ps](img/showps.png)

Penjelasan:
  * **UID**: user yang menjalankan program
  * **PID**: process IDnya
  * **PPID**: parent PID, kalau tidak ada parent akan bernilai 0
  * **C**: CPU Util. (%)
  * **STIME**: waktu proses dijalankan
  * **TTY**: terminal yang menjalankan proses. Jika tidak ada berarti background
  * **TIME**: lamanya proses berjalan
  * **CMD**: perintah yang menjalankan proses tersebut

## 4. Menghentikan Proses
Untuk menghentikan (_terminate_) proses yang berjalan, jalankan perintah shell ```kill [options] <pid>```. Biasanya untuk menghentikan paksa suatu proses dapat menggunakan perintah ```kill -9 <pid>```. 

### Macam-Macam Signal

| Signal name | Signal value  | Effect       |
| ------------|:-------------:| -------------|
| SIGHUP      | 1             | Hangup         |
| SIGINT      | 2             | Interrupt from keyboard  |
| SIGKILL     | 9             | Kill signal   |
| SIGTERM     | 15            | Termination signal
| SIGSTOP     | 17,19,23      | Stop the process

Secara default ketika menggunakan perintah shell ```kill <pid>```, akan menggunakan ```SIGSTOP``` yang mana akan menghentikan proses namun masih dapat dilanjutkan kembali.

## 5. Membuat Proses

### **fork**
```fork``` adalah fungsi _system call_ di C untuk melakukan _spawning process_. Setelah memanggil fungsi itu, akan terdapat proses baru yang merupakan _child process_ dan mengembalikan nilai 0 untuk _child process_ dan nilai _PID_ untuk _parent process_. 

# Daemon
## 1. Pengertian Daemon
Daemon adalah suatu program yang berjalan di background secara terus menerus tanpa adanya interaksi secara langsung dengan user yang sedang aktif.

<!-- Sebuah daemon dapat berhenti karena beberapa hal. -->
## 2. Langkah Pembuatan Daemon
Ada beberapa langkah untuk membuat sebuah daemon:
### 2.1 Melakukan Fork pada Parent Process dan mematikan Parent Process
Langkah pertama adalah membuat sebuah parent process dan memunculkan child process dengan melakukan `fork()`. Kemudian bunuh parent process agar sistem operasi mengira bahwa proses telah selesai.

```
pid_t pid;        // Variabel untuk menyimpan PID

pid = fork();     // Menyimpan PID dari Child Process

/* Keluar saat fork gagal
 * (nilai variabel pid < 0) */
if (pid < 0) {
  exit(EXIT_FAILURE);
}

/* Keluar saat fork berhasil
 * (nilai variabel pid adalah PID dari child process) */
if (pid > 0) {
  exit(EXIT_SUCCESS);
}
```

### 2.2 Mengubah Mode File dengan `umask`
Setiap file dan directory memiliki _permission_ atau izin yang mengatur siapa saja yang boleh melakukan _read, write,_ dan _execute_ pada file atau directory tersebut.

Dengan menggunakan `umask` kita dapat mengatur _permission_ dari suatu file pada saat file itu dibuat. Di sini kita mengatur nilai `umask(0)` agar kita mendapatkan akses full terhadap file yang dibuat oleh daemon.

```
umask(0);
```

### 2.3 Membuat Unique Session ID (SID)
Sebuah Child Process harus memiliki SID agar dapat berjalan. Tanpa adanya SID, Child Process yang Parent-nya sudah di-`kill` akan menjadi Orphan Process.

Untuk mendapatkan SID kita dapat menggunakan perintah `setsid()`. Perintah tersebut memiliki _return type_ yang sama dengan perintah `fork()`.

```
sid = setsid();
if (sid < 0) {
  exit(EXIT_FAILURE);
}
```

### 2.4 Mengubah Working Directory
Working directory harus diubah ke suatu directory yang pasti ada. Untuk amannya, kita akan mengubahnya ke root (/) directory karena itu adalah directory yang dijamin ada pada semua distro linux.

Untuk mengubah Working Directory, kita dapat menggunakan perintah `chdir()`.

```
if ((chdir("/")) < 0) {
  exit(EXIT_FAILURE);
}
```

### 2.5 Menutup File Descriptor Standar
Sebuah daemon tidak boleh menggunakan terminal. Oleh sebab itu kita harus _menutup_ file descriptor standar (STDIN, STDOUT, STDERR).

```
close(STDIN_FILENO);
close(STDOUT_FILENO);
close(STDERR_FILENO);
```

### 2.6 Membuat Loop Utama
Di loop utama ini lah tempat kita menuliskan inti dari program kita. Jangan lupa beri perintah `sleep()` agar loop berjalan pada suatu interval.

```
while (1) {
  // Tulis program kalian di sini

  sleep(30);
}
```

## 3. Kerangka Utama Daemon
Di bawah ini adalah kode hasil gabungan dari langkah-langkah pembuatan daemon:

```
pid_t pid;        // Variabel untuk menyimpan PID

pid = fork();     // Menyimpan PID dari Child Process

/* Keluar saat fork gagal
 * (nilai variabel pid < 0) */
if (pid < 0) {
  exit(EXIT_FAILURE);
}

/* Keluar saat fork berhasil
 * (nilai variabel pid adalah PID dari child process) */
if (pid > 0) {
  exit(EXIT_SUCCESS);
}

umask(0);

sid = setsid();
if (sid < 0) {
  exit(EXIT_FAILURE);
}

if ((chdir("/")) < 0) {
  exit(EXIT_FAILURE);
}

close(STDIN_FILENO);
close(STDOUT_FILENO);
close(STDERR_FILENO);

while (1) {
  // Tulis program kalian di sini

  sleep(30);
}
```

## Referensi
* http://www.netzmafia.de/skripten/unix/linux-daemon-howto.html
* https://www.computerhope.com/unix/uumask.htm