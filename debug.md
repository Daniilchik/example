# Debug: Knox init-container зависает

`knox-init-master` зависает после запуска `knoxcli.sh create-master`.
JVM стартует, подхватывает `JAVA_TOOL_OPTIONS`, но дальше — тишина и timeout.

## Что уже исключено

- ✅ Shell-скрипт работает, `bash -x` показывает все шаги до `exec java`
- ✅ Master secret приходит из Vault Agent (`/vault/secrets/knox-master-secret`, длина > 0)
- ✅ Java версия корректная (1.8.0_472)
- ✅ `JAVA_TOOL_OPTIONS=-Djava.security.egd=file:/dev/./urandom` подхвачен JVM
- ❌ `/dev/random` НЕ причина — флаг есть, всё равно висит

## Гипотезы что осталось

1. **DNS / hostname resolution** — `InetAddress.getLocalHost()` ждёт reverse DNS для pod hostname
2. **Native libs** из `/home/knox/ext/native` — отсутствуют или битые
3. **Java security providers** — Bouncy Castle / FIPS блокируется на инициализации
4. **Кривой knoxcli.jar** — баг в самой сборке

## Цель — снять thread dump через jstack/QUIT-signal

Это **точно** покажет на каком методе JVM висит.

---

## Шаг 1. Поднять debug-pod без Istio

В namespace с PodSecurity Standard `restricted` (наш случай — `dev-prizva`)
нельзя использовать `kubectl run` с короткими overrides — admission webhook
отвергнет pod без полного `securityContext`. Поэтому через YAML-файл.

Сохрани как `knox-debug.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: knox-debug
  namespace: dev-prizva           # ← подставь свой namespace
  annotations:
    sidecar.istio.io/inject: "false"
spec:
  restartPolicy: Never
  securityContext:
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: knox-debug
      image: docker-dev.registry-ci.delta.sbrf.ru/ci04674694/ci04674694/knox@sha256:cf6a339d24673fe6a7745ecb0fe71e36d899cfcbc5911d681f6b3a3a516c20c9
      command: ["sleep"]
      args: ["3600"]
      securityContext:
        allowPrivilegeEscalation: false
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        capabilities:
          drop:
            - ALL
        seccompProfile:
          type: RuntimeDefault
```

И запусти:

```bash
kubectl apply -f knox-debug.yaml

# Дождаться Running
kubectl get pod -n dev-prizva knox-debug -w
# Ctrl-C когда Running

# Зайти внутрь
kubectl exec -it -n dev-prizva knox-debug -- /bin/sh
```

Дальше — всё внутри pod-а.

---

## Шаг 2. Базовая проверка окружения

```sh
# Java версия и пути
java -version
echo $JAVA_HOME
echo $PATH

# Hostname резолвится?
hostname
getent hosts $(hostname) || echo "FAIL: no reverse DNS for $(hostname)"
cat /etc/hosts
cat /etc/resolv.conf

# Структура Knox
ls -la /home/knox/
ls -la /home/knox/bin/
ls -la /home/knox/ext/ 2>&1 || echo "no ext/ dir"
ls -la /home/knox/ext/native/ 2>&1 || echo "no native/ dir"

# Какие security providers?
java -XshowSettings:properties 2>&1 | grep -i security | head -10

# Содержимое knoxcli.sh
head -50 /home/knox/bin/knoxcli.sh
```

**Что искать:**
- `FAIL: no reverse DNS` → гипотеза А (DNS)
- Папка `ext/native` пуста или отсутствует → гипотеза Б (native libs)
- Подозрительные providers (FIPS, PKCS11, NSS) → гипотеза В

---

## Шаг 3. Снять thread dump

```sh
export JAVA_TOOL_OPTIONS="-Djava.security.egd=file:/dev/./urandom"

# Запуск knoxcli в background, перенаправляем stderr+stdout в файл
/home/knox/bin/knoxcli.sh create-master --master "TestMaster1234" --force </dev/null >/tmp/knox.log 2>&1 &
KNOXPID=$!
echo "Knox PID: $KNOXPID"

# Ждём 20 секунд (зависание происходит сразу, 20с с большим запасом)
sleep 20

# Проверяем, ещё жив ли процесс
if kill -0 $KNOXPID 2>/dev/null; then
    echo "=== Process $KNOXPID still running (HUNG) — taking thread dump ==="

    # SIGQUIT заставляет JVM напечатать thread dump в stderr (для нас — в /tmp/knox.log)
    kill -QUIT $KNOXPID

    # Дать JVM 2 секунды на печать dump
    sleep 2

    echo "=== Thread dump captured: ==="
    cat /tmp/knox.log
else
    echo "=== Process exited on its own ==="
    cat /tmp/knox.log
fi
```

**Самое важное** — найти в выводе **main thread**:
```
"main" #1 prio=5 os_prio=0 tid=0x... nid=0x... runnable [0x...]
   java.lang.Thread.State: RUNNABLE
        at java.net.Inet6AddressImpl.getLocalHostName(Native Method)     ← ВОТ ЗДЕСЬ
        at java.net.InetAddress.getLocalHost(InetAddress.java:653)
        at org.apache.knox.gateway.GatewayServer.<init>(...)
        ...
```

**Верхняя строка main thread** = точная причина зависания.

Можно отфильтровать:
```sh
grep -A 20 '"main"' /tmp/knox.log | head -30
```

---

## Шаг 4. Действия в зависимости от диагноза

### А. DNS / hostname resolution

Если thread dump показывает `Inet*Address` / `getLocalHost` / `getCanonicalHostName`:

```sh
# Workaround в debug-pod-е
echo "127.0.0.1 $(hostname).$(hostname -d 2>/dev/null) $(hostname) localhost" | sudo tee -a /etc/hosts

# Или передать Java явный hostname (требует прав root в /etc/hosts)
export JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -Djava.rmi.server.hostname=127.0.0.1"

# Повторить запуск
/home/knox/bin/knoxcli.sh create-master --master "Test1234" --force </dev/null
```

**Если помогло** → в helm-чарте добавить `hostAliases` в deployment:
```yaml
spec:
  hostAliases:
    - ip: "127.0.0.1"
      hostnames:
        - "knox.local"
        - "localhost.localdomain"
```

### Б. Native libs (ext/native)

Если thread dump показывает `System.loadLibrary` / `ClassLoader.loadLibrary0`:

```sh
ls -la /home/knox/ext/native/

# Если папки нет — создать пустую
mkdir -p /home/knox/ext/native
chmod 755 /home/knox/ext/native

# Передать пустой library.path
export JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -Djava.library.path=/tmp"

# Повторить
```

**Если помогло** → в Dockerfile добавить `mkdir -p /home/knox/ext/native`.

### В. Java security providers

Если thread dump показывает `Provider.<init>` или специфичный provider (sun.security.pkcs11, BC):

```sh
# Какой java.security файл активен
ls -la $JAVA_HOME/lib/security/java.security

# Создать минимальный properties без подозрительных providers
cat > /tmp/java.security.minimal <<EOF
security.provider.1=sun.security.provider.Sun
security.provider.2=sun.security.rsa.SunRsaSign
security.provider.3=sun.security.ec.SunEC
EOF

export JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -Djava.security.properties==/tmp/java.security.minimal"
# обрати внимание на двойное `==` — это override, не append

/home/knox/bin/knoxcli.sh create-master --master "Test1234" --force </dev/null
```

### Г. Кривой knoxcli.jar

Если ничего из вышеперечисленного — пересобрать образ из upstream Apache Knox без SDP-патчей и проверить:

```sh
# Простой тест что Knox умеет хотя бы help
java -cp /home/knox/bin/knoxcli.jar org.apache.knox.gateway.GatewayServer --help 2>&1 | head -20

# Запуск с verbose class loading
java -verbose:class -jar /home/knox/bin/knoxcli.jar create-master --master "T1234" --force 2>&1 | tail -50
# Если последняя загруженная class даст подсказку где висит
```

---

## Шаг 5. Запасной план — preset master через Secret

Если ничего не работает или нужно срочно — **обойти knoxcli.sh** вообще:

### Сгенерить master file на машине разработчика

Локально, где есть Knox jar или Java:

```bash
# Скачать knox tarball с Apache, или из тарбола вашей сборки
KNOX_LIB=/path/to/extracted/knox/lib
KNOX_DEP=/path/to/extracted/knox/dep

java -cp "$KNOX_LIB/*:$KNOX_DEP/*" \
  org.apache.knox.gateway.GatewayServer create-master \
  --master "YourRealMasterSecret123" --force

# Появится файл `data/security/master`
ls -la data/security/master
```

### Создать Secret с этим файлом

```bash
kubectl create secret generic knox-master \
  --from-file=master=./data/security/master \
  -n <namespace>
```

### Поправить deployment

Заменить наш `knox-init-master` init container на простое копирование:

```yaml
initContainers:
  - name: knox-init-master
    image: ...
    command: ["/bin/sh", "-c"]
    args:
      - |
        mkdir -p /home/knox/data/security/keystores
        cp /preset/master /home/knox/data/security/master
        chmod 600 /home/knox/data/security/master
        echo "master file installed"
    volumeMounts:
      - name: knox-data
        mountPath: /home/knox/data
      - name: knox-master-preset
        mountPath: /preset

volumes:
  - name: knox-data
    emptyDir: {}
  - name: knox-master-preset
    secret:
      secretName: knox-master
      defaultMode: 0400
```

JVM при init не запускается → проблема обходится. Gateway.sh start уже работает с готовым master file.

---

## Шаг 6. Очистка

После завершения debug:

```bash
kubectl delete -f knox-debug.yaml
# или
kubectl delete pod -n dev-prizva knox-debug
```

---

## Что прислать в команду

После thread dump:
1. **Первые 30-50 строк** `/tmp/knox.log` (включая main thread stack)
2. Вывод `ls -la /home/knox/ext/native/`
3. Вывод `getent hosts $(hostname)`
4. Версию Java: `java -version 2>&1`

С этим — за 5 минут поймём точную причину и наметим фикс.

---

## Чек-лист

- [ ] Поднял `knox-debug` pod без Istio injection
- [ ] Проверил DNS / hostname / ext/native структуру
- [ ] Запустил knoxcli.sh в background
- [ ] Снял thread dump через `kill -QUIT`
- [ ] Нашёл main thread в dump
- [ ] Применил фикс по гипотезе А/Б/В/Г
- [ ] Если не сработало → запасной план с preset master через Secret
- [ ] Удалил `knox-debug` pod после завершения
