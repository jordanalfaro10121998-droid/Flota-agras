import streamlit as st
import pandas as pd
from database import session, Registro, Stock
from reporte import generar_pdf

st.set_page_config(page_title="Agras T40 Fleet Control", layout="wide")

# Sidebar
with st.sidebar:
    st.title("🚁 Gestión Agras")
    menu = st.radio("Navegación", ["Panel de Control", "Nuevo Registro", "Inventario"])

# 1. PANEL DE CONTROL (Dashboard)
if menu == "Panel de Control":
    st.header("Resumen de Operaciones")
    
    # Métricas y Alertas
    ultimo = session.query(Registro).order_by(Registro.id.desc()).first()
    horas_actuales = ultimo.horas_vuelo if ultimo else 0
    prox_mant = 50 - (horas_actuales % 50)

    c1, c2, c3 = st.columns(3)
    c1.metric("Horas Totales", f"{horas_actuales}h")
    
    if prox_mant <= 5:
        c2.error(f"Mantenimiento en: {prox_mant}h")
    else:
        c2.success(f"Próximo chequeo: {prox_mant}h")
    
    # Tabla de registros
    st.subheader("Historial Reciente")
    regs = session.query(Registro).all()
    if regs:
        df = pd.DataFrame([(r.id, r.fecha, r.tipo, r.tecnico, r.partes) for r in regs],
                          columns=['ID', 'Fecha', 'Tipo', 'Técnico', 'Repuestos'])
        st.dataframe(df.sort_index(ascending=False), use_container_width=True)
        
        # Botón PDF para el último registro
        if st.button("Descargar Reporte PDF del Último Servicio"):
            archivo = generar_pdf(ultimo)
            with open(archivo, "rb") as f:
                st.download_button("Guardar Archivo PDF", f, file_name=archivo)
    else:
        st.info("No hay registros aún.")

# 2. NUEVO REGISTRO
elif menu == "Nuevo Registro":
    st.header("Registrar Actividad Técnica")
    with st.form("form_mant"):
        col_f1, col_f2 = st.columns(2)
        tipo = col_f1.selectbox("Tipo", ["Rutina", "Reparación", "Inspección"])
        tecnico = col_f1.text_input("Técnico")
        horas = col_f2.number_input("Horas de Vuelo Actuales", min_value=0.0, step=0.1)
        
        partes_db = [s.nombre for s in session.query(Stock).all()]
        parte_usada = col_f2.selectbox("Repuesto Utilizado", ["Ninguno"] + partes_db)
        notas = st.text_area("Notas Técnicas")
        
        enviar = st.form_submit_button("Guardar y Actualizar Stock")
        
        if enviar:
            # Guardar Registro
            nuevo = Registro(tipo=tipo, tecnico=tecnico, partes=parte_usada, horas_vuelo=horas, notas=notas)
            session.add(nuevo)
            
            # Descontar Stock
            if parte_usada != "Ninguno":
                item = session.query(Stock).filter_by(nombre=parte_usada).first()
                if item and item.cantidad > 0:
                    item.cantidad -= 1
                    st.success(f"Registro exitoso. Stock de {parte_usada} actualizado.")
                else:
                    st.warning("Pieza registrada, pero no hay stock disponible.")
            
            session.commit()
            st.rerun()

# 3. INVENTARIO
elif menu == "Inventario":
    st.header("Control de Repuestos")
    items = session.query(Stock).all()
    for i in items:
        col_i1, col_i2 = st.columns([3, 1])
        col_i1.write(f"**{i.nombre}**")
        col_i2.write(f"{i.cantidad} disp.")
        st.progress(min(i.cantidad / 10, 1.0))
