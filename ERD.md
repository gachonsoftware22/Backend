Table users {
user_id    bigint      [pk, increment]
login_id   varchar(50) [unique, not null]
password   varchar(255) [not null]
name       varchar(50) [not null]
email      varchar(100) [unique, not null]
phone      varchar(20)
birth_date date        [not null]
gender     char(1)
status     varchar(10) [not null, default: 'ACTIVE']
created_at datetime    [not null, default: `now()`]
updated_at datetime
}

Table refresh_token {
token_id   bigint      [pk, increment]
user_id    bigint      [not null, ref: > users.user_id]
token_hash varchar(255) [unique, not null]
expires_at datetime    [not null]
revoked    boolean     [not null, default: false]
created_at datetime    [not null, default: `now()`]
}

Table health {
health_id   bigint   [pk, increment]
user_id     bigint   [not null, ref: > users.user_id]
symptom     text     [not null]
history     text
note        text
record_date date     [not null]
created_at  datetime [not null, default: `now()`]
updated_at  datetime
}

Table prescription {
prescription_id   bigint      [pk, increment]
user_id           bigint      [not null, ref: > users.user_id]
prescription_date date        [not null]
hospital_name     varchar(100) [not null]
status            varchar(10) [not null, default: 'ACTIVE']
created_at        datetime    [not null, default: `now()`]
updated_at        datetime
}

Table prescription_detail {
detail_id       bigint      [pk, increment]
prescription_id bigint      [not null, ref: > prescription.prescription_id]
medicine_name   varchar(100) [not null]
dosage          varchar(50) [not null]
duration        varchar(50) [not null]
created_at      datetime    [not null, default: `now()`]
updated_at      datetime
}

Table ai_result {
result_id             int      [pk, increment]
user_id               bigint   [not null, ref: > users.user_id]
health_status         varchar(20) [not null]
summary_note          text     [not null]
potential_diseases    json
recommended_foods     json
recommended_exercises json
precautions           text
analysis_date         datetime [not null]
raw_llm_response      json
}