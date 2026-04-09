-------------------------------------------------------Tables-----------------------------------------------------------------------------------------------

CREATE TABLE "AMOUNT_DETAIL" 
   (	"AMMOUNT_ID" NUMBER(12,0) GENERATED ALWAYS AS IDENTITY MINVALUE 1 MAXVALUE 9999999999999999999999999999 INCREMENT BY 1 START WITH 1 CACHE 20 NOORDER  NOCYCLE  NOKEEP  NOSCALE  NOT NULL ENABLE, 
	"GROUP_ID" NUMBER(12,0), 
	"EXPENSE_ID" NUMBER(12,0), 
	"BORROWER_USER_ID" NUMBER(12,0), 
	"BORROW_AMOUNT" NUMBER(12,2) CONSTRAINT "NN_BRW_AMNT" NOT NULL ENABLE, 
	"LENDER_USER_ID" NUMBER(12,0), 
	"LENDING_AMOUNT" NUMBER(12,0) CONSTRAINT "NN_LND_AMNT" NOT NULL ENABLE, 
	 CONSTRAINT "PK_AMT_ID" PRIMARY KEY ("AMMOUNT_ID")
  USING INDEX  ENABLE
   ) ;

  CREATE TABLE "EXPENSE" 
   (	"EXPENSE_ID" NUMBER(12,0), 
	"GROUP_ID" NUMBER(12,0), 
	"EXPENSE_NAME" VARCHAR2(256) CONSTRAINT "NN_EXP_NAME" NOT NULL ENABLE, 
	"EXPENSE_TYPE" VARCHAR2(256) CONSTRAINT "NN_EXP_TYPE" NOT NULL ENABLE, 
	"EXPENSE_AMOUNT" NUMBER(12,2) CONSTRAINT "NN_EXP_AMNT" NOT NULL ENABLE, 
	"TRANSCATION_DATE" DATE DEFAULT sysdate, 
	"NOTE" VARCHAR2(4000), 
	 CONSTRAINT "PK_EXP_ID" PRIMARY KEY ("EXPENSE_ID")
  USING INDEX  ENABLE
   ) ;

  CREATE TABLE "GROUP_DETAILS" 
   (	"GROUP_ID" NUMBER(12,0), 
	"GROUP_NAME" VARCHAR2(256) CONSTRAINT "NN_GRP_NAME" NOT NULL ENABLE, 
	"CATEGORY" VARCHAR2(256) CONSTRAINT "NN_CAT" NOT NULL ENABLE, 
	"DESCRIPTION" VARCHAR2(4000), 
	 CONSTRAINT "PK_GRP_ID" PRIMARY KEY ("GROUP_ID")
  USING INDEX  ENABLE
   ) ;

  CREATE TABLE "USER_DETAIL" 
   (	"USER_ID" NUMBER(12,0), 
	"USER_NAME" VARCHAR2(256) CONSTRAINT "NN_USR_NAME" NOT NULL ENABLE, 
	"FIRST_NAME" CHAR(256) CONSTRAINT "NN_FIRST_NAME" NOT NULL ENABLE, 
	"LAST_NAME" CHAR(256) CONSTRAINT "NN_LST_NAME" NOT NULL ENABLE, 
	"EMAIL" VARCHAR2(256), 
	"MOBILE" NUMBER(10,0), 
	 CONSTRAINT "PK_USR_ID" PRIMARY KEY ("USER_ID")
  USING INDEX  ENABLE, 
	 CONSTRAINT "UK_USR_NAME" UNIQUE ("USER_NAME")
  USING INDEX  ENABLE
   ) ;

  CREATE TABLE "USER_GROUP" 
   (	"USR_DEL_ID" NUMBER(12,0), 
	"GROUP_ID" NUMBER(12,0), 
	"USER_ID" NUMBER(12,0), 
	 CONSTRAINT "PK_USR_DEL_ID" PRIMARY KEY ("USR_DEL_ID")
  USING INDEX  ENABLE
   ) ;

-------------------------------------------------CONSTRAINT------------------------------------------------------------------------------------------------

  ALTER TABLE "AMOUNT_DETAIL" ADD CONSTRAINT "FK_BRWR_USR_ID" FOREIGN KEY ("BORROWER_USER_ID")
	  REFERENCES "USER_DETAIL" ("USER_ID") ENABLE;
  ALTER TABLE "AMOUNT_DETAIL" ADD CONSTRAINT "FK_EXP_ID" FOREIGN KEY ("EXPENSE_ID")
	  REFERENCES "EXPENSE" ("EXPENSE_ID") ON DELETE CASCADE ENABLE;
  ALTER TABLE "AMOUNT_DETAIL" ADD CONSTRAINT "FK_GRP_ID" FOREIGN KEY ("GROUP_ID")
	  REFERENCES "GROUP_DETAILS" ("GROUP_ID") ENABLE;
  ALTER TABLE "AMOUNT_DETAIL" ADD CONSTRAINT "FK_LND_USR_ID" FOREIGN KEY ("LENDER_USER_ID")
	  REFERENCES "USER_DETAIL" ("USER_ID") ENABLE;

  ALTER TABLE "EXPENSE" ADD CONSTRAINT "FK_EXP_GRP" FOREIGN KEY ("GROUP_ID")
	  REFERENCES "GROUP_DETAILS" ("GROUP_ID") ENABLE;

  ALTER TABLE "USER_GROUP" ADD CONSTRAINT "FK_USR_GRP_GRPID" FOREIGN KEY ("GROUP_ID")
	  REFERENCES "GROUP_DETAILS" ("GROUP_ID") ON DELETE CASCADE ENABLE;
  ALTER TABLE "USER_GROUP" ADD CONSTRAINT "FK_USR_GRP_USRID" FOREIGN KEY ("USER_ID")
	  REFERENCES "USER_DETAIL" ("USER_ID") ON DELETE CASCADE ENABLE;
	  
---------------------------------------------------Functions------------------------------------------------------------------------------------------------

create or replace function get_borrower_details(p_group_id in number, p_expense_id in number) return clob is
    v_clob CLOB;
BEGIN
    v_clob := ('<ul class="ul-sql">');
    FOR i IN (
        SELECT
            borrower_user_id,
            borrow_amount
        FROM
            amount_detail
        WHERE
                group_id =p_group_id
            AND
                expense_id =p_expense_id
    ) LOOP
        v_clob := v_clob || '<li class="li-sql"><strong>'
         || get_user_name(i.borrower_user_id)
         || '</strong> owes <strong>'
         || i.borrow_amount
         || '</strong></li>';
    END LOOP;

    v_clob := v_clob || '</ul>';
    return v_clob;
END get_borrower_details;
/

create or replace function get_lender_detail(p_group_id in number, p_expense_id in number) return clob is
    cursor c_payer is
        select 
        ad.lender_user_id,
        e.expense_amount,
        e.transcation_date,
        e.note
        from 
            expense e,
            amount_detail ad
         where ad.expense_id = e.expense_id
         and ad.group_id = p_group_id
         and ad.expense_id = p_expense_id
         and rownum= 1;
    
    r_payer c_payer%rowtype;
    v_clob clob;
Begin
    open c_payer;
    fetch c_payer into r_payer;
   v_clob := '<div><strong>'
        || get_user_name(r_payer.lender_user_id)
        || '</strong> paid <strong>'
        || r_payer.expense_amount
        || '</strong> on '
        || r_payer.transcation_date
        || '</div><div style="padding:8px 0px 0px 16px">'
        || r_payer.note
        || '</div>';
    close c_payer;
    return v_clob;
END get_lender_detail;
/

create or replace function get_user_name(p_user_id in user_detail.user_id%type) return varchar2 is
    v_user_name varchar(256);
Begin
    select user_name
    into v_user_name from user_detail
        where user_id = p_user_id;
    
    return v_user_name;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN 'Unknown User';

    WHEN OTHERS THEN
        RETURN 'Error';
end get_user_name;
/

------------------------------------------------------------------------------INDEX---------------------------------------------------------------------------

  CREATE UNIQUE INDEX "PK_GRP_ID" ON "GROUP_DETAILS" ("GROUP_ID") 
  ;

  CREATE UNIQUE INDEX "SYS_C00194221726" ON "DEMO_CONSTRAINT_LOOKUP" ("CONSTRAINT_NAME") 
  ;

  CREATE UNIQUE INDEX "PK_EXP_ID" ON "EXPENSE" ("EXPENSE_ID") 
  ;

  CREATE UNIQUE INDEX "PK_USR_DEL_ID" ON "USER_GROUP" ("USR_DEL_ID") 
  ;

  CREATE UNIQUE INDEX "PK_AMT_ID" ON "AMOUNT_DETAIL" ("AMMOUNT_ID") 
  ;

  CREATE UNIQUE INDEX "SYS_C00194221699" ON "DEMO_TAGS" ("ID") 
  ;

  CREATE UNIQUE INDEX "UK_USR_NAME" ON "USER_DETAIL" ("USER_NAME") 
  ;

  CREATE UNIQUE INDEX "PK_USR_ID" ON "USER_DETAIL" ("USER_ID") 
  ;

------------------------------------------------------------------------SEQUENCE--------------------------------------------------------------------------------------

   CREATE SEQUENCE  "SEQ_AMT_DEL_ID"  MINVALUE 1 MAXVALUE 9999999999999999999999999999 INCREMENT BY 1 START WITH 1 CACHE 20 NOORDER  NOCYCLE  NOKEEP  NOSCALE  GLOBAL ;

   CREATE SEQUENCE  "SEQ_EXPENSE_ID"  MINVALUE 1 MAXVALUE 999999 INCREMENT BY 1 START WITH 221 CACHE 20 NOORDER  NOCYCLE  NOKEEP  NOSCALE  GLOBAL ;

   CREATE SEQUENCE  "SEQ_GROUP_ID"  MINVALUE 1 MAXVALUE 9999999999999999999999999999 INCREMENT BY 1 START WITH 21 CACHE 20 NOORDER  NOCYCLE  NOKEEP  NOSCALE  GLOBAL ;

   CREATE SEQUENCE  "SEQ_USR_DEL_ID"  MINVALUE 1 MAXVALUE 9999999999999999999999999999 INCREMENT BY 1 START WITH 41 CACHE 10 NOORDER  NOCYCLE  NOKEEP  NOSCALE  GLOBAL ;

   CREATE SEQUENCE  "SEQ_USR_ID"  MINVALUE 1 MAXVALUE 9999999999999999999999999999 INCREMENT BY 1 START WITH 81 CACHE 20 NOORDER  NOCYCLE  NOKEEP  NOSCALE  GLOBAL ;
