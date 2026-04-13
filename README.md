import streamlit as st
import pandas as pd
import plotly.express as px
from sqlalchemy import create_engine
import numpy as np

# =========================
# CONFIGURACIÓN GENERAL
# =========================
st.set_page_config(
    layout="wide",
    page_title="Control y seguimiento Sala Situacional",
    page_icon="📊"
)

st.title("📊 7.ª Consulta Sala Situacional del Consejo Federal de Gobierno")
st.caption("Monitoreo de actividades, seguimiento jerárquico y análisis visual de resultados")

# =========================
# CONFIGURACIÓN BD
# =========================
DB_CONFIG = {
    "host": "localhost",
    "port": "5432",
    "database": "postgres",
    "user": "postgres",
    "password": "25286022",
}

TABLA_SALA_SITUACIONAL = "public.sala_situacional"
TABLA_ESTADOS_PROYECTOS = "public.estados_proyectos"

# =========================
# FUNCIONES AUXILIARES
# =========================
def limpiar_texto(valor):
    if pd.isna(valor):
        return None
    valor = str(valor).strip()
    if valor == "" or valor.lower() in ["nan", "none", "null", "nat"]:
        return None
    return valor

def create_sample_data():
    np.random.seed(42)
    fechas = pd.date_range(start="2026-03-01", end="2026-03-20", periods=80)

    tipos = ["Llamadas Obpp"] * 55 + ["Mensajes Obpp"] * 25
    detalles_llamadas = ["Por cargar"] * 35 + ["Pagados"] * 20
    detalles_mensajes = ["Mensajes Pagados"] * 25
    detalles = detalles_llamadas + detalles_mensajes

    estados_disponibles = ["Bolívar", "Monagas", "Anzoátegui", "Sucre", "Delta Amacuro"]
    municipios_por_estado = {
        "Bolívar": ["Angostura del Orinoco", "Caroní", "Piar"],
        "Monagas": ["Maturín", "Ezequiel Zamora", "Caripe"],
        "Anzoátegui": ["Simón Bolívar", "Sotillo", "Anaco"],
        "Sucre": ["Sucre", "Bermúdez", "Arismendi"],
        "Delta Amacuro": ["Tucupita", "Pedernales", "Antonio Díaz"],
    }

    estados = np.random.choice(estados_disponibles, 80)
    municipios = [np.random.choice(municipios_por_estado[estado]) for estado in estados]

    info_recibida = []
    for i in range(80):
        if i < 55:
            info_recibida.append(np.random.choice(["Sí", "No"], p=[0.65, 0.35]))
        else:
            info_recibida.append("No aplica")

    observaciones = []
    for i in range(80):
        if i < 55:
            observaciones.append(
                np.random.choice(
                    [
                        "Proyecto cargado con éxito",
                        "No contactado",
                        "Pendiente por cargar proyecto",
                        "Desconocimiento del proceso",
                        "Informado para realizar la carga",
                        "Convocados para firma de convenio",
                        "Convenio firmado",
                    ],
                    p=[0.22, 0.20, 0.14, 0.08, 0.06, 0.18, 0.12],
                )
            )
        else:
            observaciones.append("Comunicación vía mensaje")

    data = {
        "tipo_actividad": tipos,
        "detalles_actividad": detalles,
        "fecha_inicio": np.random.choice(fechas, 80),
        "nombre_organizacion": [f"Organización {i+1}" for i in range(80)],
        "codigo_proyecto": [f"COD-{i+1:05d}" for i in range(80)],
        "informacion_recibida": info_recibida,
        "observaciones": observaciones,
        "estado": estados,
        "municipio": municipios,
    }

    df = pd.DataFrame(data)
    df["fecha_inicio"] = pd.to_datetime(df["fecha_inicio"])
    return df

# =========================
# CARGA DE DATOS
# =========================
@st.cache_data
def load_data():
    try:
        engine = create_engine(
            f"postgresql://{DB_CONFIG['user']}:{DB_CONFIG['password']}@"
            f"{DB_CONFIG['host']}:{DB_CONFIG['port']}/{DB_CONFIG['database']}"
        )

        query = f"""
        SELECT
            ss.*,
            ep.estado,
            ep.municipio
        FROM {TABLA_SALA_SITUACIONAL} ss
        LEFT JOIN {TABLA_ESTADOS_PROYECTOS} ep
            ON UPPER(TRIM(CAST(ss.codigo_proyecto AS TEXT))) = UPPER(TRIM(CAST(ep.codigo_proyecto AS TEXT)))
        """

        df = pd.read_sql(query, engine)

        columnas_esperadas = [
            "tipo_actividad",
            "detalles_actividad",
            "fecha_inicio",
            "nombre_organizacion",
            "codigo_proyecto",
            "informacion_recibida",
            "observaciones",
            "estado",
            "municipio",
        ]

        for col in columnas_esperadas:
            if col not in df.columns:
                df[col] = None

        columnas_texto = [
            "tipo_actividad",
            "detalles_actividad",
            "nombre_organizacion",
            "codigo_proyecto",
            "informacion_recibida",
            "observaciones",
            "estado",
            "municipio",
        ]

        for col in columnas_texto:
            df[col] = df[col].apply(limpiar_texto)

        df["fecha_inicio"] = pd.to_datetime(df["fecha_inicio"], errors="coerce")

        # Regla para mensajes
        mask_mensajes = df["tipo_actividad"] == "Mensajes Obpp"
        df.loc[mask_mensajes, "detalles_actividad"] = "Mensajes Pagados"
        df.loc[mask_mensajes, "informacion_recibida"] = "No aplica"
        df.loc[mask_mensajes, "observaciones"] = "Comunicación vía mensaje"

        # Mantener estructura actual
        columnas_para_mostrar = [
            "tipo_actividad",
            "detalles_actividad",
            "informacion_recibida",
            "observaciones",
            "estado",
            "municipio",
            "nombre_organizacion",
            "codigo_proyecto",
        ]

        for col in columnas_para_mostrar:
            df[col] = df[col].fillna("Sin especificar")

        return df

    except Exception as e:
        st.warning(f"No se pudo conectar a la base de datos. Se usarán datos de ejemplo. Detalle: {e}")
        df = create_sample_data()

        columnas_para_mostrar = [
            "tipo_actividad",
            "detalles_actividad",
            "informacion_recibida",
            "observaciones",
            "estado",
            "municipio",
            "nombre_organizacion",
            "codigo_proyecto",
        ]

        for col in columnas_para_mostrar:
            df[col] = df[col].fillna("Sin especificar")

        return df

# =========================
# ESTADO DE SESIÓN
# =========================
if "nivel_jerarquia" not in st.session_state:
    st.session_state.nivel_jerarquia = 0
    st.session_state.filtros = []
    st.session_state.categoria_seleccionada = None

NIVELES_TITULO = [
    "Distribución por Tipo de Actividad",
    "Distribución por Detalles de Actividad",
    "Distribución por Información Recibida",
    "Distribución por Estado",
    "Distribución por Municipio",
    "Distribución por Observaciones",
]

ICONOS_NIVEL = ["🎯", "📋", "ℹ️", "🗺️", "🏛️", "📝"]
NOMBRES_NIVEL = ["Tipo Actividad", "Detalles", "Información", "Estado", "Municipio", "Observaciones"]

def reset_to_start():
    st.session_state.nivel_jerarquia = 0
    st.session_state.filtros = []
    st.session_state.categoria_seleccionada = None

def go_back():
    if st.session_state.nivel_jerarquia > 0:
        st.session_state.nivel_jerarquia -= 1
        if st.session_state.filtros:
            st.session_state.filtros.pop()
        st.session_state.categoria_seleccionada = None

# =========================
# CARGAR DATA
# =========================
df = load_data()
df = df.dropna(subset=["fecha_inicio"]).copy()

# =========================
# OPCIONES DE FILTRO TERRITORIAL
# =========================
estados_disponibles = sorted([
    e for e in df["estado"].dropna().astype(str).unique()
    if e.strip() != ""
])

municipios_disponibles_base = sorted([
    m for m in df["municipio"].dropna().astype(str).unique()
    if m.strip() != ""
])

# =========================
# SIDEBAR
# =========================
st.sidebar.header("📅 Filtros Globales")

min_date = df["fecha_inicio"].min().date()
max_date = df["fecha_inicio"].max().date()

col_fecha1, col_fecha2 = st.sidebar.columns(2)
with col_fecha1:
    fecha_inicio = st.date_input(
        "Fecha inicial",
        min_date,
        min_value=min_date,
        max_value=max_date,
    )

with col_fecha2:
    fecha_fin = st.date_input(
        "Fecha final",
        max_date,
        min_value=min_date,
        max_value=max_date,
    )

fecha_inicio_dt = pd.Timestamp(fecha_inicio)
fecha_fin_dt = pd.Timestamp(fecha_fin) + pd.Timedelta(days=1) - pd.Timedelta(seconds=1)

st.sidebar.caption(f"📆 Mostrando: {fecha_inicio} al {fecha_fin}")

st.sidebar.markdown("---")
st.sidebar.subheader("🗺️ Filtros territoriales")

estado_filtro = st.sidebar.selectbox(
    "Estado",
    options=["Todos"] + estados_disponibles,
    index=0
)

if estado_filtro != "Todos":
    municipios_disponibles = sorted(
        df[df["estado"] == estado_filtro]["municipio"].dropna().astype(str).unique()
    )
else:
    municipios_disponibles = municipios_disponibles_base

municipio_filtro = st.sidebar.selectbox(
    "Municipio",
    options=["Todos"] + municipios_disponibles,
    index=0
)

if estado_filtro != "Todos" or municipio_filtro != "Todos":
    st.sidebar.markdown("### 📌 Filtro territorial activo")
    st.sidebar.markdown(f"**Estado:** {estado_filtro}")
    st.sidebar.markdown(f"**Municipio:** {municipio_filtro}")

st.sidebar.markdown("---")

col_sb1, col_sb2 = st.sidebar.columns(2)
with col_sb1:
    st.button("🔄 Inicio", on_click=reset_to_start, use_container_width=True, type="primary")
with col_sb2:
    st.button(
        "⬅️ Atrás",
        on_click=go_back,
        use_container_width=True,
        disabled=(st.session_state.nivel_jerarquia == 0),
    )

if st.session_state.filtros:
    st.sidebar.markdown("---")
    st.sidebar.markdown("### 🧭 Ruta actual")
    for i, filtro in enumerate(st.session_state.filtros):
        st.sidebar.markdown(f"**{ICONOS_NIVEL[i]} {NOMBRES_NIVEL[i]}:** {filtro}")

st.sidebar.markdown("---")
st.sidebar.markdown("### 📖 Guía de navegación")
st.sidebar.markdown(
    """
**🎯 Nivel 0:** Tipo de Actividad  
**📋 Nivel 1:** Detalles de Actividad  
**ℹ️ Nivel 2:** Información Recibida  
**🗺️ Nivel 3:** Estado  
**🏛️ Nivel 4:** Municipio  
**📝 Nivel 5:** Observaciones  
"""
)

st.sidebar.markdown("---")
st.sidebar.markdown("### 💾 Base de datos")
st.sidebar.markdown(f"**Host:** {DB_CONFIG['host']}")
st.sidebar.markdown(f"**Database:** {DB_CONFIG['database']}")
st.sidebar.markdown(f"**Tabla:** {TABLA_SALA_SITUACIONAL}")

# =========================
# FILTROS
# =========================
df_filtered = df[
    (df["fecha_inicio"] >= fecha_inicio_dt) &
    (df["fecha_inicio"] <= fecha_fin_dt)
].copy()

if estado_filtro != "Todos":
    df_filtered = df_filtered[df_filtered["estado"] == estado_filtro]

if municipio_filtro != "Todos":
    df_filtered = df_filtered[df_filtered["municipio"] == municipio_filtro]

df_hierarchy = df_filtered.copy()

for i, filtro_valor in enumerate(st.session_state.filtros):
    if i == 0:
        df_hierarchy = df_hierarchy[df_hierarchy["tipo_actividad"] == filtro_valor]
    elif i == 1:
        df_hierarchy = df_hierarchy[df_hierarchy["detalles_actividad"] == filtro_valor]
    elif i == 2:
        df_hierarchy = df_hierarchy[df_hierarchy["informacion_recibida"] == filtro_valor]
    elif i == 3:
        df_hierarchy = df_hierarchy[df_hierarchy["estado"] == filtro_valor]
    elif i == 4:
        df_hierarchy = df_hierarchy[df_hierarchy["municipio"] == filtro_valor]

if st.session_state.categoria_seleccionada:
    categoria = st.session_state.categoria_seleccionada
    st.session_state.filtros.append(categoria)
    if st.session_state.nivel_jerarquia < 5:
        st.session_state.nivel_jerarquia += 1
    st.session_state.categoria_seleccionada = None
    st.rerun()

# =========================
# MÉTRICAS
# =========================
st.subheader("📈 Métricas Generales")

llamadas = len(df_hierarchy[df_hierarchy["tipo_actividad"] == "Llamadas Obpp"])
mensajes = len(df_hierarchy[df_hierarchy["tipo_actividad"] == "Mensajes Obpp"])
total_registros = len(df_hierarchy)

if total_registros > 0:
    info_si = df_hierarchy[
        df_hierarchy["informacion_recibida"].astype(str).str.contains("S[íi]", na=False, case=False)
    ]
    pct_info = (len(info_si) / total_registros) * 100
else:
    pct_info = 0

m1, m2, m3, m4, m5 = st.columns(5)
m1.metric("📊 Total registros", total_registros)
m2.metric("📞 Llamadas", llamadas)
m3.metric("💬 Mensajes", mensajes)
m4.metric("✅ % Info recibida", f"{pct_info:.1f}%")
m5.metric(f"{ICONOS_NIVEL[st.session_state.nivel_jerarquia]} Nivel", NOMBRES_NIVEL[st.session_state.nivel_jerarquia])

st.markdown("---")

# =========================
# TÍTULO DEL NIVEL ACTUAL
# =========================
st.subheader(f"{ICONOS_NIVEL[st.session_state.nivel_jerarquia]} {NIVELES_TITULO[st.session_state.nivel_jerarquia]}")

if st.session_state.nivel_jerarquia == 0:
    columna_mostrar = "tipo_actividad"
elif st.session_state.nivel_jerarquia == 1:
    columna_mostrar = "detalles_actividad"
elif st.session_state.nivel_jerarquia == 2:
    columna_mostrar = "informacion_recibida"
elif st.session_state.nivel_jerarquia == 3:
    columna_mostrar = "estado"
elif st.session_state.nivel_jerarquia == 4:
    columna_mostrar = "municipio"
else:
    columna_mostrar = "observaciones"

# =========================
# VISUALIZACIÓN PRINCIPAL
# =========================
if len(df_hierarchy) > 0:
    df_grafico = df_hierarchy.copy()

    # Solo ocultar "Sin especificar" cuando el nivel sea Estado o Municipio
    if columna_mostrar in ["estado", "municipio"]:
        df_grafico = df_grafico[df_grafico[columna_mostrar].astype(str) != "Sin especificar"]

    if len(df_grafico) > 0:
        conteo = df_grafico[columna_mostrar].astype(str).value_counts().reset_index()
        conteo.columns = ["categoria", "cantidad"]
        conteo["porcentaje"] = (conteo["cantidad"] / conteo["cantidad"].sum() * 100).round(1)
        conteo = conteo.head(15)

        col_grafico, col_resumen = st.columns([3, 1])

        with col_grafico:
            if st.session_state.nivel_jerarquia in [0, 1, 2]:
                fig = px.pie(
                    conteo,
                    values="cantidad",
                    names="categoria",
                    hole=0.5,
                    title=f"{NIVELES_TITULO[st.session_state.nivel_jerarquia]} | Total: {len(df_grafico)} registros",
                    color_discrete_sequence=px.colors.qualitative.Set3
                )
                fig.update_traces(
                    textposition="inside",
                    textinfo="percent+label",
                    hovertemplate="<b>%{label}</b><br>Cantidad: %{value}<br>Porcentaje: %{percent}<extra></extra>"
                )
                fig.update_layout(
                    height=520,
                    margin=dict(t=60, b=40, l=20, r=20),
                    legend=dict(
                        orientation="h",
                        yanchor="bottom",
                        y=-0.25,
                        xanchor="center",
                        x=0.5
                    )
                )
            else:
                conteo = conteo.sort_values("cantidad", ascending=True)

                fig = px.bar(
                    conteo,
                    x="cantidad",
                    y="categoria",
                    orientation="h",
                    text="cantidad",
                    title=f"{NIVELES_TITULO[st.session_state.nivel_jerarquia]} | Total: {len(df_grafico)} registros",
                    color="cantidad",
                    color_continuous_scale="Blues"
                )
                fig.update_traces(
                    textposition="outside",
                    hovertemplate="<b>%{y}</b><br>Cantidad: %{x}<extra></extra>"
                )
                fig.update_layout(
                    height=560,
                    margin=dict(t=60, b=40, l=20, r=40),
                    xaxis_title="Cantidad",
                    yaxis_title="Categoría",
                    coloraxis_showscale=False
                )

            st.plotly_chart(fig, use_container_width=True)

        with col_resumen:
            st.markdown("### Resumen")
            st.dataframe(
                conteo[["categoria", "cantidad", "porcentaje"]],
                use_container_width=True,
                height=320
            )

            st.markdown("### Navegación")
            st.caption("Selecciona una categoría")

            for _, row in conteo.iterrows():
                categoria = row["categoria"]
                cantidad = row["cantidad"]
                porcentaje = row["porcentaje"]

                if st.button(
                    f"{categoria} ({cantidad} | {porcentaje}%)",
                    key=f"btn_{categoria}_{st.session_state.nivel_jerarquia}",
                    use_container_width=True
                ):
                    st.session_state.categoria_seleccionada = categoria
                    st.rerun()
    else:
        st.warning("⚠️ No hay datos para mostrar en este nivel.")
else:
    st.warning("⚠️ No hay datos para mostrar con los filtros actuales")
    if st.button("🔄 Resetear todos los filtros", use_container_width=True):
        reset_to_start()
        st.rerun()

# =========================
# EVOLUCIÓN TEMPORAL
# =========================
st.markdown("---")
st.subheader("📅 Evolución de actividades por fecha")

if len(df_hierarchy) > 0:
    df_tiempo = df_hierarchy.copy()
    df_tiempo["fecha_dia"] = df_tiempo["fecha_inicio"].dt.date

    evolucion = (
        df_tiempo.groupby(["fecha_dia", "tipo_actividad"])
        .size()
        .reset_index(name="cantidad")
    )

    fig_tiempo = px.line(
        evolucion,
        x="fecha_dia",
        y="cantidad",
        color="tipo_actividad",
        markers=True,
        title="Comportamiento diario de actividades"
    )

    fig_tiempo.update_layout(
        height=420,
        margin=dict(t=50, b=30, l=20, r=20),
        xaxis_title="Fecha",
        yaxis_title="Cantidad"
    )

    st.plotly_chart(fig_tiempo, use_container_width=True)
else:
    st.info("No hay datos para mostrar en la evolución temporal.")

# =========================
# LLAMADAS POR ESTADO Y MUNICIPIO
# =========================
st.markdown("---")
st.subheader("📞 Distribución territorial de llamadas")

df_llamadas = df_hierarchy[df_hierarchy["tipo_actividad"] == "Llamadas Obpp"].copy()

# Aquí sí quitamos "Sin especificar" solo para estas gráficas
df_llamadas = df_llamadas[
    (df_llamadas["estado"].astype(str) != "Sin especificar") &
    (df_llamadas["municipio"].astype(str) != "Sin especificar")
]

if len(df_llamadas) > 0:
    col_estado, col_municipio = st.columns(2)

    with col_estado:
        llamadas_estado = (
            df_llamadas["estado"]
            .astype(str)
            .value_counts()
            .reset_index()
        )
        llamadas_estado.columns = ["estado", "cantidad"]
        llamadas_estado = llamadas_estado.head(15).sort_values("cantidad", ascending=True)

        fig_estado = px.bar(
            llamadas_estado,
            x="cantidad",
            y="estado",
            orientation="h",
            text="cantidad",
            title="Llamadas por estado",
            color="cantidad",
            color_continuous_scale="Blues"
        )
        fig_estado.update_traces(
            textposition="outside",
            hovertemplate="<b>%{y}</b><br>Llamadas: %{x}<extra></extra>"
        )
        fig_estado.update_layout(
            height=450,
            margin=dict(t=50, b=30, l=20, r=30),
            xaxis_title="Cantidad de llamadas",
            yaxis_title="Estado",
            coloraxis_showscale=False
        )
        st.plotly_chart(fig_estado, use_container_width=True)

    with col_municipio:
        llamadas_municipio = (
            df_llamadas["municipio"]
            .astype(str)
            .value_counts()
            .reset_index()
        )
        llamadas_municipio.columns = ["municipio", "cantidad"]
        llamadas_municipio = llamadas_municipio.head(20).sort_values("cantidad", ascending=True)

        fig_municipio = px.bar(
            llamadas_municipio,
            x="cantidad",
            y="municipio",
            orientation="h",
            text="cantidad",
            title="Llamadas por municipio",
            color="cantidad",
            color_continuous_scale="Blues"
        )
        fig_municipio.update_traces(
            textposition="outside",
            hovertemplate="<b>%{y}</b><br>Llamadas: %{x}<extra></extra>"
        )
        fig_municipio.update_layout(
            height=450,
            margin=dict(t=50, b=30, l=20, r=30),
            xaxis_title="Cantidad de llamadas",
            yaxis_title="Municipio",
            coloraxis_showscale=False
        )
        st.plotly_chart(fig_municipio, use_container_width=True)

    st.markdown("### Resumen territorial de llamadas")
    r1, r2, r3 = st.columns(3)
    r1.metric("📞 Total llamadas territoriales", len(df_llamadas))
    r2.metric("🗺️ Estados con llamadas", df_llamadas["estado"].nunique())
    r3.metric("🏛️ Municipios con llamadas", df_llamadas["municipio"].nunique())
else:
    st.info("No hay llamadas para mostrar en la distribución por estado y municipio.")

# =========================
# TABLA DETALLE
# =========================
with st.expander("📋 Ver detalles de los registros actuales", expanded=False):
    columnas_mostrar = [
        "tipo_actividad",
        "detalles_actividad",
        "estado",
        "municipio",
        "nombre_organizacion",
        "codigo_proyecto",
        "informacion_recibida",
        "observaciones",
        "fecha_inicio",
    ]

    columnas_existentes = [col for col in columnas_mostrar if col in df_hierarchy.columns]

    if len(df_hierarchy) > 0:
        st.dataframe(
            df_hierarchy[columnas_existentes].sort_values("fecha_inicio", ascending=False),
            use_container_width=True,
            height=320
        )

        e1, e2, e3, e4 = st.columns(4)
        e1.metric("🏢 Organizaciones únicas", df_hierarchy["nombre_organizacion"].nunique())
        e2.metric("📌 Proyectos únicos", df_hierarchy["codigo_proyecto"].nunique())
        e3.metric("🗺️ Estados únicos", df_hierarchy["estado"].nunique())
        e4.metric("🏛️ Municipios únicos", df_hierarchy["municipio"].nunique())

        fecha_min = df_hierarchy["fecha_inicio"].min().date()
        fecha_max = df_hierarchy["fecha_inicio"].max().date()
        st.metric("📆 Rango fechas", f"{fecha_min} a {fecha_max}")
    else:
        st.info("📭 No hay registros para mostrar")

# =========================
# DEPURACIÓN
# =========================
with st.expander("🔧 Info de depuración", expanded=False):
    st.write("Nivel actual:", st.session_state.nivel_jerarquia)
    st.write("Filtros aplicados:", st.session_state.filtros)
    st.write("Columna visualizada:", columna_mostrar)
    st.write("Total registros:", len(df_hierarchy))
    st.write("Categoría seleccionada:", st.session_state.categoria_seleccionada)
    st.write("Filtro estado global:", estado_filtro)
    st.write("Filtro municipio global:", municipio_filtro)

    st.markdown("### Registros sin coincidencia territorial")
    sin_territorio = df[
        (df["estado"] == "Sin especificar") | (df["municipio"] == "Sin especificar")
    ]
    st.write(f"Total sin territorio identificado: {len(sin_territorio)}")

    if len(sin_territorio) > 0:
        st.dataframe(
            sin_territorio[["codigo_proyecto", "nombre_organizacion", "estado", "municipio"]].head(50),
            use_container_width=True
        )
