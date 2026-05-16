# Laporan Resmi Praktikum Sistem Operasi Modul 4

---

## Identitas

| Nama                    | NRP        |
| ----------------------- | ---------- |
| Arjunina Maqbulin Usman | 5027251007 |

---

# Soal 1 : Save Asisten Kenz      
Pada soal ini, diminta membuat sebuah filesystem virtual menggunakan FUSE (Filesystem in Userspace) dengan nama program `kenz_rescue.c`. Program harus melakukan passthrough filesystem, yaitu seluruh file asli dari source directory tetap bisa diakses melalui mount directory tanpa mengubah isi file tersebut. Program ini juga harus membuat 1 file virtual tambahan bernama `tujuan.txt` yang isinya merupakan gabungan koordinat ritual dari ketujuh file.

## Penyelesaian 
Setelah mengambil amba_files.zip dan berhasil diunzip lanjut membuat program FUSE dengan nama `kenz_rescue.c`.           
Isi dan penjelasan `kenz_rescue`:      
```c
#define FUSE_USE_VERSION 28
#include <fuse.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <dirent.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/stat.h>

static const char *dirpath =
"/home/dell_7420arjuninamu/SISOP-4-2026-IT-007/soal_1/amba_files";

/* =========================
   GETATTR
========================= */
static int x_getattr(const char *path, struct stat *stbuf)
{
    int res;
    char fpath[1000];

    memset(stbuf, 0, sizeof(struct stat));

    /* root */
    if (strcmp(path, "/") == 0) {
        stbuf->st_mode = S_IFDIR | 0755;
        stbuf->st_nlink = 2;
        return 0;
    }

    /* virtual file */
    if (strcmp(path, "/tujuan.txt") == 0) {
        stbuf->st_mode = S_IFREG | 0644;
        stbuf->st_nlink = 1;
        stbuf->st_size = 1000;
        return 0;
    }

    sprintf(fpath, "%s%s", dirpath, path);

    res = lstat(fpath, stbuf);

    if (res == -1)
        return -errno;

    return 0;
}

/* =========================
   READDIR
========================= */
static int x_readdir(const char *path,
                     void *buf,
                     fuse_fill_dir_t filler,
                     off_t offset,
                     struct fuse_file_info *fi)
{
    DIR *dp;
    struct dirent *de;
    char fpath[1000];

    (void) offset;
    (void) fi;

    sprintf(fpath, "%s", dirpath);

    dp = opendir(fpath);

    if (dp == NULL)
        return -errno;

    while ((de = readdir(dp)) != NULL) {

        struct stat st;

        memset(&st, 0, sizeof(st));

        st.st_ino = de->d_ino;
        st.st_mode = de->d_type << 12;

        filler(buf, de->d_name, &st, 0);
    }

    closedir(dp);

    /* tambah file virtual */
    filler(buf, "tujuan.txt", NULL, 0);

    return 0;
}

/* =========================
   OPEN
========================= */
static int x_open(const char *path,
                  struct fuse_file_info *fi)
{
    char fpath[1000];

    if (strcmp(path, "/tujuan.txt") == 0)
        return 0;

    sprintf(fpath, "%s%s", dirpath, path);

    int fd = open(fpath, fi->flags);

    if (fd == -1)
        return -errno;

    close(fd);

    return 0;
}

/* =========================
   READ
========================= */
static int x_read(const char *path,
                  char *buf,
                  size_t size,
                  off_t offset,
                  struct fuse_file_info *fi)
{
    (void) fi;

    /* =========================
       tujuan.txt
    ========================= */
    if (strcmp(path, "/tujuan.txt") == 0) {

        char hasil[4096] = "";

        for (int i = 1; i <= 7; i++) {

            char filepath[1000];
            char line[1024];

            sprintf(filepath, "%s/%d.txt", dirpath, i);

            FILE *fp = fopen(filepath, "r");

            if (!fp)
                continue;

            while (fgets(line, sizeof(line), fp)) {

                if (strstr(line, "KOORD:")) {

                    char *p = strchr(line, ':');

                    if (p) {

                        p++;

                        while (*p == ' ')
                            p++;

                        p[strcspn(p, "\n")] = 0;

                        strcat(hasil, p);

                        if (i != 7)
                            strcat(hasil, "_");
                    }

                    break;
                }
            }

            fclose(fp);
        }

        strcat(hasil, "\n");

        int len = strlen(hasil);

        if (offset >= len)
            return 0;

        if (offset + size > len)
            size = len - offset;

        memcpy(buf, hasil + offset, size);

        return size;
    }

    /* =========================
       passthrough file asli
    ========================= */
    char fpath[1000];

    sprintf(fpath, "%s%s", dirpath, path);

    int fd = open(fpath, O_RDONLY);

    if (fd == -1)
        return -errno;

    int res = pread(fd, buf, size, offset);

    if (res == -1)
        res = -errno;

    close(fd);

    return res;
}

/* =========================
   OPERATIONS
========================= */
static struct fuse_operations x_oper = {
    .getattr = x_getattr,
    .readdir = x_readdir,
    .open = x_open,
    .read = x_read,
};

/* =========================
   MAIN
========================= */
int main(int argc, char *argv[])
{
    umask(0);
    return fuse_main(argc, argv, &x_oper, NULL);
}
```
