# fastapi-start

# 프로젝트 구조 
project/   
├── main.py   
├── models.py   
├── database.py   
├── crud.py   
├── schemas.py   
└── requirements.txt   

# 프로젝트 실행 방법
1. 프로젝트 디렉토리로 이동
2. `pip install -r requirements.txt` 명령어로 필요한 패키지 설치
3. `uvicorn main:app --reload` 명령어로 서버 실행

localhost:8000/docs 에서 확인 가능


# 참고 자료
https://fastapi.tiangolo.com/tutorial/first-steps/


# 전체 통합 코드 
```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.orm import sessionmaker, declarative_base, Session
from pydantic import BaseModel
from typing import List

# Database Configuration: 데이터베이스 설정
DATABASE_URL = "sqlite:///./users.db"
engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Database Models: 데이터베이스 모델
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    email = Column(String, unique=True, index=True)
    full_name = Column(String)

# Pydantic Models (Schemas)
class UserBase(BaseModel):
    username: str
    email: str
    full_name: str

class UserCreate(UserBase):
    password: str

class User(UserBase):
    id: int

    class Config:
        orm_mode = True

# Database Dependency: 데이터베이스 연결을 위한 의존성 주입
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# CRUD Operations: 데이터베이스 조회, 생성, 수정, 삭제 작업
def get_user_by_username(db: Session, username: str):
    return db.query(User).filter(User.username == username).first()

def get_user_by_email(db: Session, email: str):
    return db.query(User).filter(User.email == email).first()

def get_users(db: Session, skip: int = 0, limit: int = 10):
    return db.query(User).offset(skip).limit(limit).all()

def create_user(db: Session, user: UserCreate):
    db_user = User(
        username=user.username, 
        email=user.email, 
        full_name=user.full_name
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

# FastAPI App: FastAPI 앱 생성
app = FastAPI()

# Create tables: 데이터베이스 테이블 생성
Base.metadata.create_all(bind=engine)

# API Endpoints: API 엔드포인트 생성
@app.post("/users/", response_model=User)
def create_user_endpoint(user: UserCreate, db: Session = Depends(get_db)):
    db_user = get_user_by_email(db, email=user.email)
    if db_user:
        raise HTTPException(status_code=400, detail="Email already registered")
    return create_user(db=db, user=user)

@app.get("/users/", response_model=List[User])
def read_users(skip: int = 0, limit: int = 10, db: Session = Depends(get_db)):
    users = get_users(db, skip=skip, limit=limit)
    return users

@app.get("/users/{username}", response_model=User)
def read_user(username: str, db: Session = Depends(get_db)):
    user = get_user_by_username(db, username=username)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user
