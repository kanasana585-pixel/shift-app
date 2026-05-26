import streamlit as st
import pandas as pd
import calendar
from ortools.sat.python import cp_model
import io

# ==========================================
# 初期設定・定数
# ==========================================
st.set_page_config(page_title="シフト自動作成 V8", layout="wide")

STAFF_LIST = ["鎌田", "藤井", "鰐渕", "下畑", "錦織", "杉田", "牧野", "小木", "大谷", "飯田", "オーブリー", "西田", "三田村", "木村", "高村", "奥田", "寺前"]
SHIFTS = ["休", "有休", "研修", "夜10", "こす早10", "こす早8", "こす早日", "ゆき早10", "ゆき早8", "ゆき早日", "パ早4", "こす遅10", "こす遅13", "ゆき遅10", "ゆき遅13", "こす遅8", "ゆき遅8", "日10", "パ4", "日8", "時短6"]

# 初期化
if "conditions" not in st.session_state:
    st.session_state.conditions = []
if "holidays" not in st.session_state:
    st.session_state.holidays = {}
if "selected_month" not in st.session_state:
    st.session_state.selected_month = 6

def get_default_holidays(month, days_in_month):
    default_base = {"鎌田":10, "藤井":10, "鰐渕":10, "下畑":10, "錦織":10, "杉田":10, "牧野":9, "大谷":10, "西田":9, "三田村":9, "木村":9, "高村":12, "寺前":9}
    h_10 = {1:14, 2:12, 3:13, 4:13, 5:14, 6:13, 7:13, 8:14, 9:13, 10:14, 11:13, 12:14}
    diff_8h = {1:1, 2:-1, 3:0, 4:0, 5:1, 6:0, 7:0, 8:1, 9:0, 10:1, 11:0, 12:1}
    
    holidays = {}
    for name in STAFF_LIST:
        if name in ["小木", "飯田", "オーブリー"]:
            holidays[name] = h_10[month]
        elif name == "奥田":
            h = 0
            for d in range(1, days_in_month + 1):
                w = calendar.weekday(2026, month, d)
                if w in [6, 2, 5]: # 日(6), 水(2), 土(5) ※calendarは月曜が0
                    h += 1
            holidays[name] = h
        elif month == 1 and name in ["牧野", "三田村", "木村", "寺前"]:
            holidays[name] = 10
        else:
            holidays[name] = default_base[name] + diff_8h[month]
    return holidays

# ==========================================
# 画面UI
# ==========================================
st.title("📅 シフト自動作成アプリ (V8: 早日・こすゆき分離版)")

# 1. 月選択
month = st.selectbox("対象の月 (2026年)", list(range(1, 13)), index=st.session_state.selected_month - 1)
days_in_month = calendar.monthrange(2026, month)[1]

if month != st.session_state.selected_month:
    st.session_state.selected_month = month
    st.session_state.holidays = get_default_holidays(month, days_in_month)
    st.session_state.conditions = [] # 月変更でリセット
    st.rerun()

if not st.session_state.holidays:
    st.session_state.holidays = get_default_holidays(month, days_in_month)

# 2. タブ構成
tab1, tab2, tab3 = st.tabs(["📌 条件カレンダー（確認・追加・削除）", "👤 公休数設定", "🚀 シフト自動計算"])

# ---- タブ1: 条件設定 ----
with tab1:
    col_add, col_del = st.columns(2)
    with col_add:
        st.subheader("条件の追加")
        with st.form("add_cond_form"):
            c1, c2, c3, c4 = st.columns(4)
            c_name = c1.selectbox("スタッフ", STAFF_LIST)
            c_day = c2.selectbox("日付", ["All"] + list(map(str, range(1, days_in_month + 1))))
            c_type = c3.selectbox("種類", ["固定", "禁止"])
            c_shift = c4.selectbox("シフト", SHIFTS)
            if st.form_submit_button("追加する"):
                exists = any(c for c in st.session_state.conditions if c['name']==c_name and c['day']==c_day and c['shift']==c_shift and c['type']==c_type)
                if not exists:
                    st.session_state.conditions.append({"name": c_name, "day": c_day, "type": c_type, "shift": c_shift})
                    st.rerun()
                else:
                    st.warning("既に同じ条件が登録されています。")

    with col_del:
        st.subheader("条件の削除")
        if st.session_state.conditions:
            cond_strs = [f"{c['name']} - {c['day']}日 - {c['type']}: {c['shift']}" for c in st.session_state.conditions]
            del_idx = st.selectbox("削除する条件を選択", range(len(cond_strs)), format_func=lambda x: cond_strs[x])
            if st.button("削除する"):
                st.session_state.conditions.pop(del_idx)
                st.rerun()
        else:
            st.write("設定されている条件はありません。")
    
    st.subheader("条件カレンダー")
    # カレンダー用のデータフレーム作成
    df_cal = pd.DataFrame(index=STAFF_LIST, columns=["全日"] + list(map(str, range(1, days_in_month + 1))))
    df_cal.fillna("", inplace=True)
    
    for c in st.session_state.conditions:
        col_name = "全日" if c['day'] == "All" else c['day']
        mark = "📌" if c['type'] == "固定" else "🚫"
        text = f"{mark}{c['shift']}"
        if df_cal.at[c['name'], col_name] == "":
            df_cal.at[c['name'], col_name] = text
        else:
            df_cal.at[c['name'], col_name] += f", {text}"
            
    st.dataframe(df_cal, height=600, use_container_width=True)

# ---- タブ2: 公休設定 ----
with tab2:
    st.subheader("公休数の調整")
    df_h = pd.DataFrame(list(st.session_state.holidays.items()), columns=["スタッフ", "公休数"])
    edited_df = st.data_editor(df_h, hide_index=True)
    if st.button("公休数を保存"):
        st.session_state.holidays = dict(zip(edited_df["スタッフ"], edited_df["公休数"]))
        st.success("保存しました！")

# ---- タブ3: 自動計算 ----
with tab3:
    st.subheader("シフト自動計算の実行")
    st.write("設定した条件と公休数をもとに、AIがすべてのルールを満たすシフトを計算します。（計算には最大2分かかります）")
    
    if st.button("🚀 計算スタート", type="primary"):
        with st.spinner("AIが数百万の組み合わせから最適なシフトを計算中です..."):
            # OR-Toolsモデル構築
            model = cp_model.CpModel()
            x = {}
            for e in STAFF_LIST:
                for d in range(days_in_month):
                    for s in SHIFTS:
                        x[(e, d, s)] = model.NewBoolVar(f"x_{e}_{d}_{s}")
            
            for e in STAFF_LIST:
                for d in range(days_in_month):
                    model.AddExactlyOne(x[(e, d, s)] for s in SHIFTS)
                # 公休設定
                model.Add(sum(x[(e, d, "休")] for d in range(days_in_month)) == st.session_state.holidays[e])
                # 有休数・研修数設定
                yukyu_c = sum(1 for c in st.session_state.conditions if c['name']==e and c['type']=="固定" and c['shift']=="有休")
                kenshu_c = sum(1 for c in st.session_state.conditions if c['name']==e and c['type']=="固定" and c['shift']=="研修")
                model.Add(sum(x[(e, d, "有休")] for d in range(days_in_month)) == yukyu_c)
                model.Add(sum(x[(e, d, "研修")] for d in range(days_in_month)) == kenshu_c)

            # 条件適用
            fixed_dict = {}
            for c in st.session_state.conditions:
                if c['type'] == "固定" and c['day'] != "All":
                    d_idx = int(c['day']) - 1
                    model.Add(x[(c['name'], d_idx, c['shift'])] == 1)
                    fixed_dict[(c['name'], d_idx)] = c['shift']
                elif c['type'] == "禁止":
                    if c['day'] == "All":
                        for d_idx in range(days_in_month):
                            model.Add(x[(c['name'], d_idx, c['shift'])] == 0)
                    else:
                        d_idx = int(c['day']) - 1
                        model.Add(x[(c['name'], d_idx, c['shift'])] == 0)

            # --- 基本ルール群 ---
            js_first_day = calendar.monthrange(2026, month)[0] # 月曜=0...日曜=6
            py_first_day = (js_first_day + 1) % 7 # Python側(Colab版)は日曜=6, 月曜=0だったため合わせる

            # 奥田シフト
            for d in range(days_in_month):
                if ("奥田", d) in fixed_dict: continue
                weekday = (d + py_first_day) % 7
                if weekday == 0: model.Add(x[("奥田", d, "パ早4")] == 1)
                elif weekday in [1, 3, 4]: model.Add(x[("奥田", d, "パ4")] == 1)
                else: model.Add(x[("奥田", d, "休")] == 1)
            
            for e in STAFF_LIST:
                if e != "奥田":
                    for d in range(days_in_month):
                        model.Add(x[(e, d, "パ早4")] == 0)
                        model.Add(x[(e, d, "パ4")] == 0)

            # 夜勤明け・連勤防止
            for e in STAFF_LIST:
                for d in range(days_in_month - 1):
                    model.AddImplication(x[(e, d, "夜10")], x[(e, d+1, "休")])
                for d in range(days_in_month - 4):
                    model.Add(sum(x[(e, d+i, "休")] + x[(e, d+i, "有休")] for i in range(5)) >= 1)
            
            # 遅出・早日の翌日早出禁止
            late_shifts = ["こす遅13", "こす遅10", "ゆき遅13", "ゆき遅10", "こす早日", "ゆき早日"]
            early_shifts = ["こす早10", "こす早8", "ゆき早10", "ゆき早8", "パ早4", "こす早日", "ゆき早日"]
            for e in STAFF_LIST:
                for d in range(days_in_month - 1):
                    for s_late in late_shifts:
                        for s_early in early_shifts:
                            model.AddImplication(x[(e, d, s_late)], x[(e, d+1, s_early)].Not())

            # 10H組 / 8H組の縛り
            target_8h_count = 2 if month in [3, 7] else 1
            shifts_8h_for_10h = ["日8", "こす早8", "ゆき早8", "こす遅8", "ゆき遅8"]
            allowed_10h_staff_shifts = ["こす遅10", "ゆき遅10", "日10", "こす早10", "ゆき早10", "夜10", "休", "有休", "日8", "こす早8", "ゆき早8", "こす遅8", "ゆき遅8"]
            for e in ["小木", "飯田", "オーブリー"]:
                model.Add(sum(x[(e, d, s)] for d in range(days_in_month) for s in shifts_8h_for_10h) == target_8h_count)
                for d in range(days_in_month):
                    if (e, d) in fixed_dict: continue
                    for s in SHIFTS:
                        if s not in allowed_10h_staff_shifts:
                            model.Add(x[(e, d, s)] == 0)
            
            for e in ["鎌田", "藤井", "鰐渕", "下畑", "錦織", "杉田", "牧野", "大谷"]:
                for d in range(days_in_month):
                    if (e, d) in fixed_dict: continue
                    for s in ["こす早10", "ゆき早10", "こす遅10", "ゆき遅10", "日10"]:
                        model.Add(x[(e, d, s)] == 0)

            # 個人別縛り
            for d in range(days_in_month):
                if ("牧野", d) not in fixed_dict:
                    for s in SHIFTS:
                        if s not in ["こす早8", "ゆき早8", "こす遅8", "ゆき遅8", "休", "有休"]:
                            model.Add(x[("牧野", d, s)] == 0)
            
            model.Add(sum(x[("高村", d, "こす早8")] for d in range(days_in_month)) <= 4)
            for d in range(days_in_month):
                if ("高村", d) not in fixed_dict:
                    for s in SHIFTS:
                        if s not in ["こす早8", "日8", "休", "有休"]: model.Add(x[("高村", d, s)] == 0)

            for e in STAFF_LIST:
                if e in ["木村", "三田村", "寺前"]:
                    for d in range(days_in_month):
                        if (e, d) not in fixed_dict:
                            for s in SHIFTS:
                                if s not in ["時短6", "休", "有休"]: model.Add(x[(e, d, s)] == 0)
                else:
                    for d in range(days_in_month):
                        model.Add(x[(e, d, "時短6")] == 0)

            for d in range(days_in_month):
                if ("西田", d) not in fixed_dict:
                    for s in SHIFTS:
                        if s not in ["日8", "夜10", "ゆき早8", "ゆき遅8", "こす早日", "ゆき早日", "休", "有休"]:
                            model.Add(x[("西田", d, s)] == 0)

            # --- V8 早日分離＆相殺ロジック ---
            for d in range(days_in_month):
                f_night = sum(1 for c in st.session_state.conditions if c['type']=="固定" and c['day']==str(d+1) and c['shift']=="夜10")
                f_k_o1 = sum(1 for c in st.session_state.conditions if c['type']=="固定" and c['day']==str(d+1) and c['shift'] in ["こす遅10", "こす遅13"])
                f_y_o1 = sum(1 for c in st.session_state.conditions if c['type']=="固定" and c['day']==str(d+1) and c['shift'] in ["ゆき遅10", "ゆき遅13"])
                f_k_h = sum(1 for c in st.session_state.conditions if c['type']=="固定" and c['day']==str(d+1) and c['shift'] in ["こす早10", "こす早8"])
                f_y_h = sum(1 for c in st.session_state.conditions if c['type']=="固定" and c['day']==str(d+1) and c['shift'] in ["ゆき早10", "ゆき早8", "パ早4"])
                f_k_o2 = sum(1 for c in st.session_state.conditions if c['type']=="固定" and c['day']==str(d+1) and c['shift'] in ["こす遅8", "日10"])
                f_y_o2 = sum(1 for c in st.session_state.conditions if c['type']=="固定" and c['day']==str(d+1) and c['shift'] in ["ゆき遅8", "日10"])
                
                weekday = (d + py_first_day) % 7
                if weekday == 0: f_y_h += 1
                
                model.Add(sum(x[(e, d, "夜10")] for e in STAFF_LIST) == max(2, f_night))
                model.Add(sum(x[(e, d, "こす遅10")] + x[(e, d, "こす遅13")] for e in STAFF_LIST) == max(1, f_k_o1))
                model.Add(sum(x[(e, d, "ゆき遅10")] + x[(e, d, "ゆき遅13")] for e in STAFF_LIST) == max(1, f_y_o1))
                
                kosu_hayahi = sum(x[(e, d, "こす早日")] for e in STAFF_LIST)
                yuki_hayahi = sum(x[(e, d, "ゆき早日")] for e in STAFF_LIST)
                model.Add(kosu_hayahi + yuki_hayahi <= 2)
                
                kosu_haya = sum(x[(e, d, "こす早10")] + x[(e, d, "こす早8")] for e in STAFF_LIST)
                yuki_haya = sum(x[(e, d, "ゆき早10")] + x[(e, d, "ゆき早8")] + x[(e, d, "パ早4")] for e in STAFF_LIST)
                
                if f_k_h >= 1: model.Add(kosu_haya == f_k_h)
                else: model.Add(kosu_haya + kosu_hayahi <= 1)
                    
                if f_y_h >= 1: model.Add(yuki_haya == f_y_h)
                else: model.Add(yuki_haya + yuki_hayahi <= 1)
                    
                model.Add(kosu_haya + yuki_haya + kosu_hayahi + yuki_hayahi == max(2, f_k_h + f_y_h))
                
                kosu_oso2 = sum(x[(e, d, "こす遅8")] + x[(e, d, "日10")] for e in STAFF_LIST)
                yuki_oso2 = sum(x[(e, d, "ゆき遅8")] + x[(e, d, "日10")] for e in STAFF_LIST)
                
                model.Add(kosu_oso2 + kosu_hayahi >= max(1, f_k_o2))
                model.Add(yuki_oso2 + yuki_hayahi >= max(1, f_y_o2))
                
                total_oso2 = sum(x[(e, d, "こす遅8")] + x[(e, d, "ゆき遅8")] + x[(e, d, "日10")] for e in STAFF_LIST)
                model.Add(total_oso2 + kosu_hayahi + yuki_hayahi == max(2, f_k_o2 + f_y_o2))

            # 計算実行
            solver = cp_model.CpSolver()
            solver.parameters.max_time_in_seconds = 120.0
            status = solver.Solve(model)

            if status == cp_model.OPTIMAL or status == cp_model.FEASIBLE:
                st.success("✨ シフトの計算が完了しました！")
                
                schedule = []
                for e in STAFF_LIST:
                    row = {"氏名": e}
                    work_days = 0
                    for d in range(days_in_month):
                        for s in SHIFTS:
                            if solver.Value(x[(e, d, s)]) == 1:
                                display_shift = "早日" if s in ["こす早日", "ゆき早日"] else s
                                row[str(d+1)] = display_shift
                                if s not in ["休", "有休"]: work_days += 1
                    row["休の数"] = st.session_state.holidays[e]
                    row["有休"] = sum(1 for d in range(days_in_month) if solver.Value(x[(e, d, "有休")]) == 1)
                    row["業務数"] = work_days
                    schedule.append(row)
                
                summary_rows = {
                    "夜勤": ["夜10"], "こす早": ["こす早10", "こす早8"], "ゆき早": ["ゆき早10", "ゆき早8"], "パ早4": ["パ早4"], 
                    "こす早日": ["こす早日"], "ゆき早日": ["ゆき早日"], "時短": ["時短6"], "こす遅1": ["こす遅10", "こす遅13"], 
                    "ゆき遅1": ["ゆき遅10", "ゆき遅13"], "こす遅2": ["こす遅8", "日10"], "ゆき遅2": ["ゆき遅8", "日10"]
                }
                for label, target_shifts in summary_rows.items():
                    row = {"氏名": label}
                    for d in range(days_in_month):
                        count = sum(solver.Value(x[(e, d, s)]) for e in STAFF_LIST for s in target_shifts)
                        row[str(d+1)] = count
                    schedule.append(row)
                
                df_out = pd.DataFrame(schedule)
                
                # CSV出力の準備
                csv = df_out.to_csv(index=False, encoding="utf-8-sig").encode("utf-8-sig")
                st.download_button(
                    label="📥 シフト表(CSV)をダウンロード",
                    data=csv,
                    file_name=f"shift_2026年{month}月.csv",
                    mime="text/csv",
                    type="primary"
                )
                
                st.dataframe(df_out)
            else:
                st.error("⚠️ エラー：指定された条件が厳しすぎるため、シフトを組めませんでした。固定・禁止条件や公休数を緩和してください。")
