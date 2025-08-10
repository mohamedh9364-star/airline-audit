import streamlit as st
import pandas as pd
import io

st.set_page_config(page_title="مراجعة كشوفات العملاء - شركات الطيران", layout="wide")

st.title("✈️ مراجعة كشوفات العملاء لشركات الطيران")
st.markdown("ارفع ملفات كشف الشركة وكشف العميل، وسيقوم النظام بالمقارنة، التلوين، واكتشاف التكرارات.")

# رفع الملفات
company_file = st.file_uploader("📂 ارفع كشف الشركة", type=["xlsx", "csv"])
client_file = st.file_uploader("📂 ارفع كشف العميل", type=["xlsx", "csv"])

# دالة قراءة الملفات
def load_file(file):
    if file.name.endswith(".csv"):
        return pd.read_csv(file)
    else:
        return pd.read_excel(file)

if company_file and client_file:
    company_df = load_file(company_file)
    client_df = load_file(client_file)

    st.subheader("📄 بيانات الشركة")
    st.dataframe(company_df.head())

    st.subheader("📄 بيانات العميل")
    st.dataframe(client_df.head())

    # اختيار الأعمدة
    st.sidebar.header("⚙️ الإعدادات")
    company_pnr_col = st.sidebar.selectbox("عمود رقم الحجز في الشركة", company_df.columns)
    company_amount_col = st.sidebar.selectbox("عمود المبلغ في الشركة", company_df.columns)

    client_pnr_col = st.sidebar.selectbox("عمود رقم الحجز في العميل", client_df.columns)
    client_amount_col = st.sidebar.selectbox("عمود المبلغ في العميل", client_df.columns)

    # تنظيف الـ PNR
    company_df[company_pnr_col] = company_df[company_pnr_col].astype(str).str.strip().str.upper()
    client_df[client_pnr_col] = client_df[client_pnr_col].astype(str).str.strip().str.upper()

    # الدمج للمقارنة
    merged_df = pd.merge(
        company_df[[company_pnr_col, company_amount_col]],
        client_df[[client_pnr_col, client_amount_col]],
        left_on=company_pnr_col,
        right_on=client_pnr_col,
        how="outer",
        suffixes=("_شركة", "_عميل")
    )

    # إضافة حالة المطابقة
    def check_match(row):
        if pd.isna(row[company_amount_col + "_شركة"]):
            return "❌ غير موجود في الشركة"
        elif pd.isna(row[client_amount_col + "_عميل"]):
            return "❌ غير موجود في العميل"
        elif row[company_amount_col + "_شركة"] == row[client_amount_col + "_عميل"]:
            return "✅ متطابق"
        else:
            return "⚠️ غير متطابق"

    merged_df["الحالة"] = merged_df.apply(check_match, axis=1)

    # اكتشاف التكرارات
    duplicates = merged_df[merged_df.duplicated(subset=[company_pnr_col], keep=False)]
    if not duplicates.empty:
        st.warning("⚠️ تم اكتشاف تكرارات في أرقام الحجز")
        st.dataframe(duplicates)

    # عرض النتيجة
    st.subheader("📊 النتيجة النهائية")
    st.dataframe(merged_df)

    # تحميل النتيجة كملف Excel
    towrite = io.BytesIO()
    merged_df.to_excel(towrite, index=False, engine='openpyxl')
    towrite.seek(0)

    st.download_button(
        label="💾 تحميل التقرير النهائي",
        data=towrite,
        file_name="تقرير_المقارنة.xlsx",
        mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
    )
