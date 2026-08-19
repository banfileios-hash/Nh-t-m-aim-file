#include <jni.h>
#include <dlfcn.h>
#include <sys/mman.h>
#include <stdint.h>
#include <string.h>
#include <pthread.h>
#include <unistd.h>
#include <math.h>
#include <android/log.h>

#define LOG_TAG "BulletStraight"
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__)

#define OFFSET_BULLET_SPEED     0x0012A4B0
#define OFFSET_BULLET_GRAVITY   0x0012A4B4
#define OFFSET_BULLET_DROP      0x0012A4B8
#define OFFSET_BULLET_SPREAD    0x0012A4BC
#define OFFSET_BULLET_RANGE     0x0012A4C0
#define OFFSET_FUNC_BULLET      0x00046000
#define OFFSET_FUNC_UPDATE      0x00045C30
#define OFFSET_FUNC_WEAPON      0x00045E80

void* g_base = NULL;
int g_running = 1;

void patch_memory(uintptr_t abs_addr, uint8_t* data, size_t len) {
    uintptr_t page_start = abs_addr & ~0xFFF;
    mprotect((void*)page_start, 0x1000, PROT_READ | PROT_WRITE | PROT_EXEC);
    memcpy((void*)abs_addr, data, len);
    __builtin___clear_cache((char*)abs_addr, (char*)(abs_addr + len));
}

void* bullet_thread(void* arg) {
    uintptr_t base = (uintptr_t)arg;
    while (g_running) {
        float* bullet_speed = (float*)(base + OFFSET_BULLET_SPEED);
        float* bullet_gravity = (float*)(base + OFFSET_BULLET_GRAVITY);
        float* bullet_drop = (float*)(base + OFFSET_BULLET_DROP);
        float* bullet_spread = (float*)(base + OFFSET_BULLET_SPREAD);
        float* bullet_range = (float*)(base + OFFSET_BULLET_RANGE);

        *bullet_speed = 9999.0f;
        *bullet_gravity = 0.0f;
        *bullet_drop = 0.0f;
        *bullet_spread = 0.0f;
        *bullet_range = 1000.0f;

        uint8_t straight_code[] = {
            0x00, 0x48, 0x00, 0x44,
            0x00, 0x60
        };
        patch_memory(base + OFFSET_FUNC_BULLET, straight_code, sizeof(straight_code));

        usleep(10000);
    }
    return NULL;
}

void patch_bullet(uintptr_t base) {
    uint8_t zero_code[] = {0x00, 0x20, 0x00, 0x60};
    uint8_t max_code[] = {0x00, 0x48, 0x00, 0x44, 0x00, 0x60};
    uint8_t nop_code[] = {0x00, 0xBF, 0x00, 0xBF};

    patch_memory(base + OFFSET_BULLET_SPEED, max_code, sizeof(max_code));
    patch_memory(base + OFFSET_BULLET_GRAVITY, zero_code, sizeof(zero_code));
    patch_memory(base + OFFSET_BULLET_DROP, zero_code, sizeof(zero_code));
    patch_memory(base + OFFSET_BULLET_SPREAD, zero_code, sizeof(zero_code));
    patch_memory(base + OFFSET_FUNC_BULLET, nop_code, sizeof(nop_code));

    LOGD("Bullet straight patches applied");
}

extern "C" JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved) {
    void* base = dlopen("libmain.so", RTLD_NOW);
    if (!base) base = dlopen("libgame.so", RTLD_NOW);
    if (!base) base = dlopen("libunity.so", RTLD_NOW);
    g_base = base;

    LOGD("Bullet Straight loaded at base: 0x%lx", (uintptr_t)base);

    patch_bullet((uintptr_t)base);

    pthread_t t;
    pthread_create(&t, NULL, bullet_thread, (void*)base);
    pthread_detach(t);

    LOGD("Bullet Straight active: dan thang, khong roi, khong trong luc");
    return JNI_VERSION_1_6;
}
