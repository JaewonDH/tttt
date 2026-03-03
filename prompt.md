## B/E 개발 기술 스택
* FastApi
* sqlalchemy
* python 3.12 버전 사용 
* 오라클 DB 사용 

## Database 접속 정보
* Host:     localhost
* Port:     1521
* Service:  FREEPDB1
* Username: myuser
* Password: mypassword


### 개발 준수 사항
* 주석은 한글로 추가해줘 
* 확장성을 고려해서 소스 구조를 설계 필요
* 도메인 기반의 프로젝트 구조로 설계 필요 
* DB 세션은 동기,비동기 모두 사용할 수 있도록 제공 


### 테이블 생성 규칙 
* ID는 UUID로 생성 
* 상태나 유형같은 데이터들은 코드 테이블을 만들어서 관리할꺼야


## 상세 시나리오

### 내부 권한
- Agent 시스템 접속한 이후에 사용할 수 있는 권한이 존재하면 다음과 같다. Admin, Agent Owner, Agent개발자가 있다 
- Admin은 Agent 카드 신청 및 삭제 시 승인 하는 관리자 
- Agent Owner는 Agent 관리자이며 Agent 카드 생성 후 Agent 상세 내용을 수정 및 삭제, Agent 개발자를 추가 및 변경 할 수 있다. 
- Agent 카드를 신청한 사람이 Agent Owner 권한을 갖게 된다. 
- Agent 개발자는 Agent 카드 생성 후 Agent 상세 내용을 수정 및 삭제, Agent 개발자를 추가 및 변경 할 수 있다. 

### 승인 기능 
- 사용자가 Agent 신청을 할 수 있다.
- 사용자가 Agent 이름, Agent 설명, 
- Agent System 페이지에 진입 후 Agent 카드를 신청할 수 있다.
- Agent 카드를 신청한 사람이 신청한 Agent Owner 권한을 갖게 

### Agent 상태 
- 승인대기 (Agent 카드 신청 후 대기 상태)
- 반려 (Agent 카드를 신청요청 후 Admin이 반려했을경우) 
- Dev( Agent 카드 신청 후 Admin이 승인한 이후)
- Open (Agent 배포 후 마켓 등록된경우) 

### Agent 카드 신청 기능 페이지
- 사용자가 Agent 신청을 할 수 있다.
- 사용자가 Agent 이름, Agent 설명, 개인 정보 동의 항목 10개를 체크 후 신청 한다. 
- Agent System 페이지에 진입 후 Agent 카드를 신청할 수 있다.

### Agent 목록 조회 페이지 
- Agent Owner거나 Agnet 개발자로 등록된 사용자를 기반으로 개인화 목록을 보여준다. 
- 생성된 Agent 카드를 조회 할 수 있는 기능이 필요 하다. 
- 페이지네이션 기능이 필요 하며 
- 조회 시 탭별로 승인대기, 반려, Open ,Dev 필터를 통해 조회 할 수 있다

### Agent 상세 페이지 
- 조회된 Agent 목록에서 하나를 선택하면 상세페이지로 진입 
- Agent 이름, 설명 등을 수정 할 수 있다.
- Agent 삭제를 할 수 있다. 삭제 시 바로 삭제 되지 않고 Admin에게 승인 요청이 후 Admin이 승인 이후에 삭제가 된다. 삭제는 소프트 삭제로 진행하여 Database에 이력이 남아야 한다.

### Admin 전체 Agent 조회 페이지 
- 전체 Agent 목록을 조회 하는 기능 
- 페이지네이션 기능이 필요 하며 
- 조회 시 탭별로 승인대기, 반려, Open ,Dev 필터를 통해 조회 할 수 있다

### Admin 승인 목록 조회 페이지 
- 사용자가 신청한 Agent 카드 생성 및 삭제 승인 요청한 목록을 조회하며 반려 목록같이 같이 보여준다. 
- 관리자가 반려 시 반려 사유를 적고 저장 할 수 있다.


## 아래 내용은 위의 시나리오를 기반으로 만든 오라클 쿼리 테이블이야 
```
/*
================================================================================
  Agent System - Oracle DB 전체 스크립트  v2
  ── 메타 코드 테이블 적용 기준 정비 ──────────────────────────────────────────
  ✅ TB_CODE_GROUP/DETAIL 사용 (4개): AGENT_STATUS_CD, ROLE_CD,
                                       REQ_TYPE_CD, REQ_STATUS_CD
      → UI 드롭다운·라벨·필터 탭에 직접 노출되고 운영 중 코드 추가 가능성 있음
  ❌ CHECK 제약만 사용 (나머지):
      · Y/N 이진 플래그 (USE_YN, DEL_YN, GRANT_YN, AGREE_YN, REQUIRED_YN)
      · SYNC_STATUS   → 외부 시스템 처리 코드, UI 비노출, 배치 하드코딩
      · PERMISSION_CD → ADMIN 단일값, 메타 테이블 의미 없음
      · CHANGE_TYPE_CD → 내부 이력 코드, 앱 Enum으로 충분
  순서: 0.DROP  1.DDL  2.SEQUENCE  3.INDEX  4.DUMMY DATA  5.검증 쿼리
================================================================================
*/

-- ============================================================
-- 0. 기존 오브젝트 정리 (재실행 대비)
-- ============================================================
BEGIN
  FOR t IN (
    SELECT table_name FROM user_tables
    WHERE table_name IN (
      'TB_AGENT_HISTORY','TB_AGENT_CONSENT','TB_APPROVAL_REQUEST',
      'TB_AGENT_MEMBER','TB_AGENT',
      'TB_CONSENT_ITEM','TB_CODE_DETAIL','TB_CODE_GROUP',
      'TB_USER_PERMISSION','TB_AGENT_SYSTEM_ACCESS','TB_USER_SYNC'
    )
  ) LOOP
    EXECUTE IMMEDIATE 'DROP TABLE ' || t.table_name || ' CASCADE CONSTRAINTS PURGE';
  END LOOP;

  FOR s IN (
    SELECT sequence_name FROM user_sequences
    WHERE sequence_name IN (
      'SEQ_ACCESS_ID','SEQ_USER_PERMISSION_ID','SEQ_AGENT_ID',
      'SEQ_AGENT_MEMBER_ID','SEQ_CONSENT_ITEM_ID','SEQ_AGENT_CONSENT_ID',
      'SEQ_APPROVAL_REQ_ID','SEQ_HISTORY_ID'
    )
  ) LOOP
    EXECUTE IMMEDIATE 'DROP SEQUENCE ' || s.sequence_name;
  END LOOP;
END;


-- ============================================================
-- 1. DDL - CREATE TABLE
-- ============================================================

-- ──────────────────────────────────────────
-- [외부 동기화] 사용자 동기화
-- ──────────────────────────────────────────
CREATE TABLE TB_USER_SYNC (
  USER_ID        VARCHAR2(50)   NOT NULL,
  USER_NM        VARCHAR2(100)  NOT NULL,
  EMAIL          VARCHAR2(200)  NOT NULL,
  DEPT_NM        VARCHAR2(200),
  EXT_SYSTEM_ID  VARCHAR2(50)   NOT NULL,
  -- CHECK 제약: 외부 시스템 배치 처리 코드, UI 미노출, 앱 Enum 관리
  SYNC_STATUS    VARCHAR2(20)   DEFAULT 'PENDING' NOT NULL
                   CONSTRAINT CK_USER_SYNC_STATUS
                   CHECK (SYNC_STATUS IN ('PENDING','SUCCESS','FAIL')),
  SYNC_DT        DATE,
  USE_YN         CHAR(1)        DEFAULT 'Y' NOT NULL  -- Y/N 이진 플래그
                   CONSTRAINT CK_USER_SYNC_USE CHECK (USE_YN IN ('Y','N')),
  REG_DT         DATE           DEFAULT SYSDATE NOT NULL,
  UPD_DT         DATE,
  CONSTRAINT PK_USER_SYNC PRIMARY KEY (USER_ID)
);
COMMENT ON TABLE  TB_USER_SYNC IS '사용자 동기화 (외부 시스템)';
COMMENT ON COLUMN TB_USER_SYNC.USER_ID       IS '사용자 ID (PK) - 외부 시스템 식별자';
COMMENT ON COLUMN TB_USER_SYNC.USER_NM       IS '사용자 이름';
COMMENT ON COLUMN TB_USER_SYNC.EMAIL         IS '이메일';
COMMENT ON COLUMN TB_USER_SYNC.DEPT_NM       IS '부서명';
COMMENT ON COLUMN TB_USER_SYNC.EXT_SYSTEM_ID IS '동기화 출처 외부 시스템 ID';
COMMENT ON COLUMN TB_USER_SYNC.SYNC_STATUS   IS '[CHECK] 동기화 상태 PENDING/SUCCESS/FAIL - 앱 Enum 관리';
COMMENT ON COLUMN TB_USER_SYNC.SYNC_DT       IS '최종 동기화 일시';
COMMENT ON COLUMN TB_USER_SYNC.USE_YN        IS '[CHECK] 사용여부 Y/N - 이진 플래그';


-- ──────────────────────────────────────────
-- [외부 동기화] Agent 시스템 접근 권한
-- ──────────────────────────────────────────
CREATE TABLE TB_AGENT_SYSTEM_ACCESS (
  ACCESS_ID     NUMBER         NOT NULL,
  USER_ID       VARCHAR2(50)   NOT NULL,
  GRANT_YN      CHAR(1)        DEFAULT 'Y' NOT NULL  -- Y/N 이진 플래그
                  CONSTRAINT CK_ACCESS_GRANT CHECK (GRANT_YN IN ('Y','N')),
  GRANT_REASON  VARCHAR2(500),
  GRANT_DT      DATE           DEFAULT SYSDATE NOT NULL,
  EXPIRE_DT     DATE,
  SYNC_DT       DATE,
  -- CHECK 제약: 외부 동기화 처리 코드, 앱 Enum 관리
  SYNC_STATUS   VARCHAR2(20)   DEFAULT 'PENDING' NOT NULL
                  CONSTRAINT CK_ACCESS_SYNC_STATUS
                  CHECK (SYNC_STATUS IN ('PENDING','SUCCESS','FAIL')),
  REG_DT        DATE           DEFAULT SYSDATE NOT NULL,
  UPD_DT        DATE,
  CONSTRAINT PK_AGENT_SYSTEM_ACCESS PRIMARY KEY (ACCESS_ID),
  CONSTRAINT FK_ACCESS_USER FOREIGN KEY (USER_ID) REFERENCES TB_USER_SYNC(USER_ID)
);
COMMENT ON TABLE  TB_AGENT_SYSTEM_ACCESS IS 'Agent 시스템 접근 권한 (외부 동기화)';
COMMENT ON COLUMN TB_AGENT_SYSTEM_ACCESS.ACCESS_ID    IS '접근권한 ID (PK)';
COMMENT ON COLUMN TB_AGENT_SYSTEM_ACCESS.USER_ID      IS '사용자 ID (FK)';
COMMENT ON COLUMN TB_AGENT_SYSTEM_ACCESS.GRANT_YN     IS '[CHECK] 접근 허용 여부 Y/N - 이진 플래그';
COMMENT ON COLUMN TB_AGENT_SYSTEM_ACCESS.EXPIRE_DT    IS '만료일시 (NULL=무기한)';
COMMENT ON COLUMN TB_AGENT_SYSTEM_ACCESS.SYNC_STATUS  IS '[CHECK] 동기화 상태 PENDING/SUCCESS/FAIL - 앱 Enum 관리';


-- ──────────────────────────────────────────
-- [내부] Admin 권한
-- ──────────────────────────────────────────
CREATE TABLE TB_USER_PERMISSION (
  USER_PERMISSION_ID  NUMBER         NOT NULL,
  USER_ID             VARCHAR2(50)   NOT NULL,
  -- CHECK 제약: ADMIN 단일값만 존재 - 메타 테이블 불필요
  PERMISSION_CD       VARCHAR2(20)   NOT NULL
                        CONSTRAINT CK_PERM_CD CHECK (PERMISSION_CD IN ('ADMIN')),
  USE_YN              CHAR(1)        DEFAULT 'Y' NOT NULL  -- Y/N 이진 플래그
                        CONSTRAINT CK_PERM_USE CHECK (USE_YN IN ('Y','N')),
  REG_DT              DATE           DEFAULT SYSDATE NOT NULL,
  REG_USER_ID         VARCHAR2(50)   NOT NULL,
  UPD_DT              DATE,
  UPD_USER_ID         VARCHAR2(50),
  CONSTRAINT PK_USER_PERMISSION PRIMARY KEY (USER_PERMISSION_ID),
  CONSTRAINT FK_PERM_USER FOREIGN KEY (USER_ID) REFERENCES TB_USER_SYNC(USER_ID)
);
COMMENT ON TABLE  TB_USER_PERMISSION IS 'Admin 내부 권한 (내부 관리 전용)';
COMMENT ON COLUMN TB_USER_PERMISSION.PERMISSION_CD IS '[CHECK] 권한 코드 - ADMIN 고정값 (단일값이므로 코드테이블 불필요)';


-- ──────────────────────────────────────────
-- [공통 코드] 그룹 마스터
--   관리 대상: AGENT_STATUS_CD, ROLE_CD, REQ_TYPE_CD, REQ_STATUS_CD
-- ──────────────────────────────────────────
CREATE TABLE TB_CODE_GROUP (
  GROUP_CD     VARCHAR2(50)   NOT NULL,
  GROUP_NM     VARCHAR2(100)  NOT NULL,
  GROUP_DESC   VARCHAR2(500),
  USE_YN       CHAR(1)        DEFAULT 'Y' NOT NULL
                 CONSTRAINT CK_CGRP_USE CHECK (USE_YN IN ('Y','N')),
  REG_DT       DATE           DEFAULT SYSDATE NOT NULL,
  REG_USER_ID  VARCHAR2(50)   NOT NULL,
  UPD_DT       DATE,
  UPD_USER_ID  VARCHAR2(50),
  CONSTRAINT PK_CODE_GROUP PRIMARY KEY (GROUP_CD)
);
COMMENT ON TABLE  TB_CODE_GROUP IS '공통 코드 그룹 마스터 (UI 노출 코드만 관리: AGENT_STATUS_CD, ROLE_CD, REQ_TYPE_CD, REQ_STATUS_CD)';
COMMENT ON COLUMN TB_CODE_GROUP.GROUP_CD IS '그룹 코드 (PK)';


-- ──────────────────────────────────────────
-- [공통 코드] 상세 값
-- ──────────────────────────────────────────
CREATE TABLE TB_CODE_DETAIL (
  GROUP_CD     VARCHAR2(50)   NOT NULL,
  CODE_VAL     VARCHAR2(50)   NOT NULL,
  CODE_NM      VARCHAR2(100)  NOT NULL,
  CODE_DESC    VARCHAR2(500),
  SORT_ORDER   NUMBER         NOT NULL,
  USE_YN       CHAR(1)        DEFAULT 'Y' NOT NULL
                 CONSTRAINT CK_CDET_USE CHECK (USE_YN IN ('Y','N')),
  REG_DT       DATE           DEFAULT SYSDATE NOT NULL,
  REG_USER_ID  VARCHAR2(50)   NOT NULL,
  UPD_DT       DATE,
  UPD_USER_ID  VARCHAR2(50),
  CONSTRAINT PK_CODE_DETAIL PRIMARY KEY (GROUP_CD, CODE_VAL),
  CONSTRAINT FK_DETAIL_GROUP FOREIGN KEY (GROUP_CD) REFERENCES TB_CODE_GROUP(GROUP_CD)
);
COMMENT ON TABLE  TB_CODE_DETAIL IS '공통 코드 상세 (UI 드롭다운·라벨에 직접 사용하는 코드만 관리)';
COMMENT ON COLUMN TB_CODE_DETAIL.GROUP_CD IS '그룹 코드 (PK, FK)';
COMMENT ON COLUMN TB_CODE_DETAIL.CODE_VAL IS '코드 값 (PK)';
COMMENT ON COLUMN TB_CODE_DETAIL.CODE_NM  IS '코드명 (UI 표시용)';


-- ──────────────────────────────────────────
-- [Agent 핵심] Agent 카드
-- ──────────────────────────────────────────
CREATE TABLE TB_AGENT (
  AGENT_ID         NUMBER          NOT NULL,
  AGENT_NM         VARCHAR2(200)   NOT NULL,
  AGENT_DESC       VARCHAR2(4000),
  -- 코드 테이블 사용: UI 탭 필터·상태 라벨 표시, 상태 추가 가능성 있음
  AGENT_STATUS_CD  VARCHAR2(20)    DEFAULT 'PENDING' NOT NULL
                     CONSTRAINT CK_AGENT_STATUS
                     CHECK (AGENT_STATUS_CD IN ('PENDING','REJECTED','DEV','OPEN','DELETE_PENDING')),
  OWNER_USER_ID    VARCHAR2(50)    NOT NULL,
  DEL_YN           CHAR(1)         DEFAULT 'N' NOT NULL  -- Y/N 이진 플래그
                     CONSTRAINT CK_AGENT_DEL CHECK (DEL_YN IN ('Y','N')),
  DEL_DT           DATE,
  DEL_USER_ID      VARCHAR2(50),
  REG_DT           DATE            DEFAULT SYSDATE NOT NULL,
  UPD_DT           DATE,
  REG_USER_ID      VARCHAR2(50)    NOT NULL,
  UPD_USER_ID      VARCHAR2(50),
  CONSTRAINT PK_AGENT PRIMARY KEY (AGENT_ID),
  CONSTRAINT FK_AGENT_OWNER FOREIGN KEY (OWNER_USER_ID) REFERENCES TB_USER_SYNC(USER_ID)
);
COMMENT ON TABLE  TB_AGENT IS 'Agent 카드 (핵심)';
COMMENT ON COLUMN TB_AGENT.AGENT_STATUS_CD IS '[CODE_TABLE] 상태 코드 - TB_CODE_DETAIL(AGENT_STATUS_CD) 참조';
COMMENT ON COLUMN TB_AGENT.DEL_YN          IS '[CHECK] 소프트 삭제 여부 Y/N - 이진 플래그';


-- ──────────────────────────────────────────
-- [Agent 핵심] 구성원 권한
-- ──────────────────────────────────────────
CREATE TABLE TB_AGENT_MEMBER (
  AGENT_MEMBER_ID  NUMBER         NOT NULL,
  AGENT_ID         NUMBER         NOT NULL,
  USER_ID          VARCHAR2(50)   NOT NULL,
  -- 코드 테이블 사용: 역할명 UI 표시, 역할 추가 가능성 있음
  ROLE_CD          VARCHAR2(20)   NOT NULL
                     CONSTRAINT CK_MEMBER_ROLE
                     CHECK (ROLE_CD IN ('AGENT_OWNER','AGENT_DEV')),
  USE_YN           CHAR(1)        DEFAULT 'Y' NOT NULL  -- Y/N 이진 플래그
                     CONSTRAINT CK_MEMBER_USE CHECK (USE_YN IN ('Y','N')),
  REG_DT           DATE           DEFAULT SYSDATE NOT NULL,
  UPD_DT           DATE,
  REG_USER_ID      VARCHAR2(50)   NOT NULL,
  UPD_USER_ID      VARCHAR2(50),
  CONSTRAINT PK_AGENT_MEMBER PRIMARY KEY (AGENT_MEMBER_ID),
  CONSTRAINT FK_MEMBER_AGENT FOREIGN KEY (AGENT_ID)  REFERENCES TB_AGENT(AGENT_ID),
  CONSTRAINT FK_MEMBER_USER  FOREIGN KEY (USER_ID)   REFERENCES TB_USER_SYNC(USER_ID),
  CONSTRAINT UQ_AGENT_MEMBER UNIQUE (AGENT_ID, USER_ID)
);
COMMENT ON TABLE  TB_AGENT_MEMBER IS 'Agent 구성원 권한 (Owner/Dev)';
COMMENT ON COLUMN TB_AGENT_MEMBER.ROLE_CD  IS '[CODE_TABLE] 역할 코드 - TB_CODE_DETAIL(ROLE_CD) 참조';
COMMENT ON COLUMN TB_AGENT_MEMBER.USE_YN   IS '[CHECK] 사용여부 Y/N - 이진 플래그';


-- ──────────────────────────────────────────
-- [동의] 개인정보 동의 항목 마스터
-- ──────────────────────────────────────────
CREATE TABLE TB_CONSENT_ITEM (
  CONSENT_ITEM_ID  NUMBER          NOT NULL,
  ITEM_NM          VARCHAR2(200)   NOT NULL,
  ITEM_DESC        VARCHAR2(1000),
  SORT_ORDER       NUMBER          NOT NULL,
  REQUIRED_YN      CHAR(1)         DEFAULT 'Y' NOT NULL  -- Y/N 이진 플래그
                     CONSTRAINT CK_CONSENT_REQ CHECK (REQUIRED_YN IN ('Y','N')),
  USE_YN           CHAR(1)         DEFAULT 'Y' NOT NULL  -- Y/N 이진 플래그
                     CONSTRAINT CK_CONSENT_USE CHECK (USE_YN IN ('Y','N')),
  REG_DT           DATE            DEFAULT SYSDATE NOT NULL,
  CONSTRAINT PK_CONSENT_ITEM PRIMARY KEY (CONSENT_ITEM_ID)
);
COMMENT ON TABLE  TB_CONSENT_ITEM IS '개인정보 동의 항목 마스터 (10개)';
COMMENT ON COLUMN TB_CONSENT_ITEM.REQUIRED_YN IS '[CHECK] 필수동의 여부 Y/N - 이진 플래그';
COMMENT ON COLUMN TB_CONSENT_ITEM.USE_YN      IS '[CHECK] 사용여부 Y/N - 이진 플래그';


-- ──────────────────────────────────────────
-- [동의] Agent 신청 동의 내역
-- ──────────────────────────────────────────
CREATE TABLE TB_AGENT_CONSENT (
  AGENT_CONSENT_ID  NUMBER          NOT NULL,
  AGENT_ID          NUMBER          NOT NULL,
  CONSENT_ITEM_ID   NUMBER          NOT NULL,
  AGREE_YN          CHAR(1)         NOT NULL  -- Y/N 이진 플래그
                      CONSTRAINT CK_AGREE_YN CHECK (AGREE_YN IN ('Y','N')),
  AGREE_DT          DATE            DEFAULT SYSDATE NOT NULL,
  USER_ID           VARCHAR2(50)    NOT NULL,
  CONSTRAINT PK_AGENT_CONSENT PRIMARY KEY (AGENT_CONSENT_ID),
  CONSTRAINT FK_CONSENT_AGENT  FOREIGN KEY (AGENT_ID)        REFERENCES TB_AGENT(AGENT_ID),
  CONSTRAINT FK_CONSENT_ITEM   FOREIGN KEY (CONSENT_ITEM_ID) REFERENCES TB_CONSENT_ITEM(CONSENT_ITEM_ID),
  CONSTRAINT FK_CONSENT_USER   FOREIGN KEY (USER_ID)         REFERENCES TB_USER_SYNC(USER_ID),
  CONSTRAINT UQ_AGENT_CONSENT  UNIQUE (AGENT_ID, CONSENT_ITEM_ID)
);
COMMENT ON TABLE  TB_AGENT_CONSENT IS 'Agent 신청 동의 내역';
COMMENT ON COLUMN TB_AGENT_CONSENT.AGREE_YN IS '[CHECK] 동의여부 Y/N - 이진 플래그';


-- ──────────────────────────────────────────
-- [승인] 승인 요청
-- ──────────────────────────────────────────
CREATE TABLE TB_APPROVAL_REQUEST (
  APPROVAL_REQ_ID  NUMBER          NOT NULL,
  AGENT_ID         NUMBER          NOT NULL,
  -- 코드 테이블 사용: 승인 목록 UI 유형 표시
  REQ_TYPE_CD      VARCHAR2(20)    NOT NULL
                     CONSTRAINT CK_REQ_TYPE CHECK (REQ_TYPE_CD IN ('CREATE','DELETE')),
  -- 코드 테이블 사용: 승인 목록 UI 상태 표시·탭 필터
  REQ_STATUS_CD    VARCHAR2(20)    DEFAULT 'PENDING' NOT NULL
                     CONSTRAINT CK_REQ_STATUS
                     CHECK (REQ_STATUS_CD IN ('PENDING','APPROVED','REJECTED')),
  REQ_USER_ID      VARCHAR2(50)    NOT NULL,
  REQ_DT           DATE            DEFAULT SYSDATE NOT NULL,
  PROCESS_USER_ID  VARCHAR2(50),
  PROCESS_DT       DATE,
  REJECT_REASON    VARCHAR2(2000),
  REG_DT           DATE            DEFAULT SYSDATE NOT NULL,
  CONSTRAINT PK_APPROVAL_REQUEST  PRIMARY KEY (APPROVAL_REQ_ID),
  CONSTRAINT FK_APPR_AGENT        FOREIGN KEY (AGENT_ID)        REFERENCES TB_AGENT(AGENT_ID),
  CONSTRAINT FK_APPR_REQ_USER     FOREIGN KEY (REQ_USER_ID)     REFERENCES TB_USER_SYNC(USER_ID),
  CONSTRAINT FK_APPR_PROC_USER    FOREIGN KEY (PROCESS_USER_ID) REFERENCES TB_USER_SYNC(USER_ID)
);
COMMENT ON TABLE  TB_APPROVAL_REQUEST IS '승인 요청 관리 (생성/삭제)';
COMMENT ON COLUMN TB_APPROVAL_REQUEST.REQ_TYPE_CD   IS '[CODE_TABLE] 요청 유형 - TB_CODE_DETAIL(REQ_TYPE_CD) 참조';
COMMENT ON COLUMN TB_APPROVAL_REQUEST.REQ_STATUS_CD IS '[CODE_TABLE] 처리 상태 - TB_CODE_DETAIL(REQ_STATUS_CD) 참조';
COMMENT ON COLUMN TB_APPROVAL_REQUEST.REJECT_REASON IS '반려 사유 (반려 시 필수)';


-- ──────────────────────────────────────────
-- [이력] Agent 변경 이력
-- ──────────────────────────────────────────
CREATE TABLE TB_AGENT_HISTORY (
  HISTORY_ID        NUMBER          NOT NULL,
  AGENT_ID          NUMBER          NOT NULL,
  -- CHECK 제약: 내부 이력 코드, UI 직접 노출 없음, 앱 Enum으로 충분
  CHANGE_TYPE_CD    VARCHAR2(30)    NOT NULL
                      CONSTRAINT CK_HIST_TYPE
                      CHECK (CHANGE_TYPE_CD IN ('CREATE','UPDATE','STATUS_CHANGE','DELETE_REQ','DELETE')),
  BEFORE_STATUS_CD  VARCHAR2(20),
  AFTER_STATUS_CD   VARCHAR2(20),
  BEFORE_AGENT_NM   VARCHAR2(200),
  AFTER_AGENT_NM    VARCHAR2(200),
  BEFORE_AGENT_DESC VARCHAR2(4000),
  AFTER_AGENT_DESC  VARCHAR2(4000),
  APPROVAL_REQ_ID   NUMBER,
  REG_DT            DATE            DEFAULT SYSDATE NOT NULL,
  REG_USER_ID       VARCHAR2(50)    NOT NULL,
  CONSTRAINT PK_AGENT_HISTORY  PRIMARY KEY (HISTORY_ID),
  CONSTRAINT FK_HIST_AGENT     FOREIGN KEY (AGENT_ID)        REFERENCES TB_AGENT(AGENT_ID),
  CONSTRAINT FK_HIST_APPR      FOREIGN KEY (APPROVAL_REQ_ID) REFERENCES TB_APPROVAL_REQUEST(APPROVAL_REQ_ID)
);
COMMENT ON TABLE  TB_AGENT_HISTORY IS 'Agent 변경 이력 (전체 보존)';
COMMENT ON COLUMN TB_AGENT_HISTORY.CHANGE_TYPE_CD IS '[CHECK] 변경유형 CREATE/UPDATE/STATUS_CHANGE/DELETE_REQ/DELETE - 앱 Enum 관리';


-- ============================================================
-- 2. SEQUENCE
-- ============================================================
CREATE SEQUENCE SEQ_ACCESS_ID          START WITH 1 INCREMENT BY 1 NOCACHE NOCYCLE;
CREATE SEQUENCE SEQ_USER_PERMISSION_ID START WITH 1 INCREMENT BY 1 NOCACHE NOCYCLE;
CREATE SEQUENCE SEQ_AGENT_ID           START WITH 1 INCREMENT BY 1 NOCACHE NOCYCLE;
CREATE SEQUENCE SEQ_AGENT_MEMBER_ID    START WITH 1 INCREMENT BY 1 NOCACHE NOCYCLE;
CREATE SEQUENCE SEQ_CONSENT_ITEM_ID    START WITH 1 INCREMENT BY 1 NOCACHE NOCYCLE;
CREATE SEQUENCE SEQ_AGENT_CONSENT_ID   START WITH 1 INCREMENT BY 1 NOCACHE NOCYCLE;
CREATE SEQUENCE SEQ_APPROVAL_REQ_ID    START WITH 1 INCREMENT BY 1 NOCACHE NOCYCLE;
CREATE SEQUENCE SEQ_HISTORY_ID         START WITH 1 INCREMENT BY 1 NOCACHE NOCYCLE;


-- ============================================================
-- 3. INDEX
-- ============================================================
CREATE INDEX IDX_USER_SYNC_EXT       ON TB_USER_SYNC            (EXT_SYSTEM_ID, SYNC_STATUS);
CREATE INDEX IDX_ACCESS_USER         ON TB_AGENT_SYSTEM_ACCESS  (USER_ID, GRANT_YN, EXPIRE_DT);
CREATE INDEX IDX_ACCESS_SYNC         ON TB_AGENT_SYSTEM_ACCESS  (SYNC_STATUS, SYNC_DT);
CREATE INDEX IDX_USER_PERM_USER      ON TB_USER_PERMISSION      (USER_ID, USE_YN);
CREATE INDEX IDX_AGENT_OWNER         ON TB_AGENT                (OWNER_USER_ID, DEL_YN);
CREATE INDEX IDX_AGENT_STATUS        ON TB_AGENT                (AGENT_STATUS_CD, DEL_YN);
CREATE INDEX IDX_AGENT_MEMBER_USER   ON TB_AGENT_MEMBER         (USER_ID, USE_YN);
CREATE INDEX IDX_AGENT_MEMBER_AGENT  ON TB_AGENT_MEMBER         (AGENT_ID, USE_YN);
CREATE INDEX IDX_CONSENT_AGENT       ON TB_AGENT_CONSENT        (AGENT_ID);
CREATE INDEX IDX_APPROVAL_STATUS     ON TB_APPROVAL_REQUEST     (REQ_STATUS_CD, REQ_TYPE_CD);
CREATE INDEX IDX_APPROVAL_AGENT      ON TB_APPROVAL_REQUEST     (AGENT_ID, REQ_STATUS_CD);
CREATE INDEX IDX_CODE_DETAIL_USE     ON TB_CODE_DETAIL          (GROUP_CD, USE_YN);
CREATE INDEX IDX_HISTORY_AGENT       ON TB_AGENT_HISTORY        (AGENT_ID, REG_DT);
```
