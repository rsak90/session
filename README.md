CREATE TABLE Sessions (
    SessionId NVARCHAR2(449) PRIMARY KEY,
    SessionKey NVARCHAR2(200) NOT NULL,
    SessionValue NCLOB,
    ExpiresAtTime TIMESTAMP NOT NULL,
    SlidingExpirationInSeconds NUMBER,
    AbsoluteExpiration TIMESTAMP,
);
