# Docker Compose로 구성한 Redis Sentinel (Redis 7.4)

Redis Sentinel 고가용성 환경을 Docker Compose로 구성한 프로젝트입니다. 이중화로 구성된 Redis 서버가 중단없이 failover 되는지 테스트가 핵심

## 구성

- Redis Master 1대 (redis:7.4)
- Redis Replica 1대 (redis:7.4)
- Redis Sentinel 3대 (redis:7.4)

**설정 정보**

- Redis 버전: 7.4
- 비밀번호: 환경변수 `REDIS_PASSWORD`로 주입(.env 파일 사용)

**주요 특징**

- 각 노드는 개별 설정 파일(conf)을 Docker 볼륨으로 마운트
- 모든 서비스는 `redisnet` 브리지 네트워크에 연결
- Sentinel은 Master를 `redis-local` 호스트명으로 모니터링
    - Docker 컨테이너 내부: `redis-local` → `redis-master` 컨테이너 (네트워크 alias)
    - 호스트 머신: `redis-local` → `127.0.0.1` (hosts 파일 설정)

## 포트 구성

- Master: `127.0.0.1:6379`
- Replica: `127.0.0.1:6380` (컨테이너 내부는 6379)
- Sentinel:
    - sentinel1: `127.0.0.1:26379`
    - sentinel2: `127.0.0.1:26380`
    - sentinel3: `127.0.0.1:26381`

## 설정 파일

- `docker-compose.yml`
- `redis-master/conf/redis.conf`
- `redis-replica/conf/redis.conf`
- `sentinel1/conf/sentinel.conf`
- `sentinel2/conf/sentinel.conf`
- `sentinel3/conf/sentinel.conf`

## 호스트 접근 제한

> 모든 포트는 `127.0.0.1`에 바인딩되어 로컬에서만 접근 가능합니다.

**동작 방식**

- Sentinel은 `mymaster`를 `redis-local:6379`로 모니터링
    - Docker 내부: `redis-local` → `redis-master` 컨테이너
    - Windows/macOS: hosts 파일로 `redis-local` → `127.0.0.1` 매핑
- Replica는 `redis-local:6379`를 Master로 설정하고, 외부에는 `127.0.0.1:6380`으로 노출

### Windows hosts 파일 설정

**관리자 권한으로 다음 파일을 열어 편집**

```
C:\Windows\System32\drivers\etc\hosts

--- 추가할 내용
127.0.0.1 redis-local
::1 redis-local
```

- 첫 번째 줄: IPv4 매핑
- 두 번째 줄: IPv6 매핑 (일부 환경에서 IPv6 우선 사용)

**저장 후 PowerShell(관리자)에서 DNS 캐시 초기화**

```powershell
ipconfig /flushdns
```

**참고**

- Spring Boot 애플리케이션은 `localhost:26379~26381`로 Sentinel에 접근
- 컨테이너 간 통신은 Docker 네트워크를 통해 이루어짐
- 호스트에서는 모두 `127.0.0.1`로 접근

## 실행 방법 

> .env로 비밀번호 주입

1. Docker Desktop(Windows/macOS) 또는 Docker Engine(Linux) 실행

2. 비밀번호 설정
    - 방법 1: 저장소 루트에 `.env` 파일 생성(권장, 커밋 금지)
      ```env
      REDIS_PASSWORD=비밀번호 입력
      ```
    - 방법 2: 일시적으로 환경변수 설정 후 실행
        - Windows PowerShell
          ```powershell
          $Env:REDIS_PASSWORD = "비밀번호 입력"
          ```
        - macOS/Linux Shell
          ```bash
          export REDIS_PASSWORD="비밀번호 입력"
          ```

3. 프로젝트 루트에서 컨테이너 시작
   ```bash
   docker compose up -d
   ```

4. 상태 확인
   ```bash
   docker compose ps
   # master는 healthy, 나머지는 running 상태면 정상
   ```

## 동작 확인

**Master 연결 테스트**

```bash
docker exec -it redis-master sh -c "redis-cli -a \"$REDIS_PASSWORD\" PING"
```

**Replica 역할 확인**

```bash
docker exec -it redis-replica sh -c "redis-cli -a \"$REDIS_PASSWORD\" INFO replication | grep role"
```

**Sentinel이 인식한 Master 주소 확인**

```bash
docker exec -it redis-sentinel-1 redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
# 초기값: redis-local 6379
```

**Failover 테스트**

1. Master 중지: `docker stop redis-master`
2. Sentinel 로그 확인: `docker logs -f redis-sentinel-1`
3. 승격 후 Master 주소 확인: `SENTINEL get-master-addr-by-name mymaster`

## Spring Boot 설정

> Spring Data Redis + Lettuce 사용 시 `application.yml` 설정:

```yaml
spring:
  data:
    redis:
      password: ${REDIS_PASSWORD:}
      sentinel:
        master: mymaster
        nodes:
          - localhost:26379
          - localhost:26380
          - localhost:26381
      timeout: 5s
```

**설정 설명**

- `password`: Redis Master/Replica 인증에 사용
- Sentinel 자체는 별도 인증 없음
- `sentinel auth-pass`는 Sentinel이 Master와 통신할 때 사용

**Docker 컨테이너에서 Spring Boot 실행 시**

- `redisnet` 네트워크에 연결하여 `redis-sentinel-1:26379` 등으로 접근
- 또는 `extra_hosts: ["redis-local:host-gateway"]` 추가

## 주요 설정

**redis-master/conf/redis.conf**

- `requirepass __REDIS_PASSWORD__`
- `appendonly yes`
- `bind 0.0.0.0`
- `protected-mode no`

**redis-replica/conf/redis.conf**

- `replicaof redis-local 6379`
- `masterauth __REDIS_PASSWORD__`
- `requirepass __REDIS_PASSWORD__`

**sentinel\*/conf/sentinel.conf**

- `sentinel monitor mymaster redis-local 6379 2`
- `sentinel auth-pass mymaster __REDIS_PASSWORD__`
- `sentinel resolve-hostnames yes`
- `sentinel announce-hostnames yes`
- `dir /data` (Sentinel 상태 저장용 디렉터리)

## Linux 사용 시 참고사항

Linux/WSL에서 실행하는 경우 해당 환경의 `/etc/hosts`에도 다음 추가

```
127.0.0.1 redis-local
::1 redis-local
```

Docker 20.10 이상에서는 `extra_hosts`로 `host.docker.internal`을 `host-gateway`에 매핑 가능합니다.

## 종료 및 정리

```bash
# 컨테이너 중지 및 제거
docker compose down

# 볼륨과 데이터까지 모두 삭제
docker compose down -v
```