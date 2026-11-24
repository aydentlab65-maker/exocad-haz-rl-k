name: Build Windows EXE

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: windows-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install pyvista pyvistaqt PyQt5 scipy numpy pyinstaller

      - name: Write app script
        shell: pwsh
        run: |
          @'
          import sys
          import numpy as np
          import pyvista as pv
          from pyvistaqt import QtInteractor
          from PyQt5.QtWidgets import (
              QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
              QPushButton, QLabel, QFileDialog, QDoubleSpinBox, QGroupBox,
              QMessageBox, QRadioButton
          )
          from scipy.interpolate import splprep, splev
          from scipy.spatial import cKDTree


          class DentalPrepApp(QMainWindow):
              def _init_(self):
                  super()._init_()
                  self.setWindowTitle("Exocad Ön Hazırlık - Marjin & Kapanış Düzenleyici")
                  self.setGeometry(100, 100, 1400, 850)

                  self.upper_mesh = None
                  self.lower_mesh = None
                  self.active_jaw = "upper"

                  self.margin_points = []
                  self.margin_actor = None
                  self.spline_actor = None

                  self.align_mode = False
                  self.upper_align_pts = []
                  self.lower_align_pts = []
                  self.align_actors = []

                  self.init_ui()

              def init_ui(self):
                  main_widget = QWidget()
                  self.setCentralWidget(main_widget)
                  layout = QHBoxLayout(main_widget)

                  control_panel = QVBoxLayout()
                  control_panel.setSpacing(12)

                  file_group = QGroupBox("1. STL Tarama Yükleme")
                  file_layout = QVBoxLayout()
                  btn_load_upper = QPushButton("Üst Çene STL Yükle")
                  btn_load_upper.clicked.connect(lambda: self.load_stl("upper"))
                  btn_load_lower = QPushButton("Alt Çene STL Yükle")
                  btn_load_lower.clicked.connect(lambda: self.load_stl("lower"))
                  file_layout.addWidget(btn_load_upper)
                  file_layout.addWidget(btn_load_lower)
                  file_group.setLayout(file_layout)
                  control_panel.addWidget(file_group)

                  align_group = QGroupBox("2. Kapanış (Oklüzyon) Hizalama")
                  align_layout = QVBoxLayout()
                  self.lbl_align_info = QLabel("Durum: Bekleniyor")
                  self.lbl_align_info.setWordWrap(True)
                  btn_start_align = QPushButton("3 Nokta Kapanış Başlat")
                  btn_start_align.clicked.connect(self.start_alignment_mode)
                  btn_run_icp = QPushButton("Seçilen Noktalarla Kapanışı Kilitle")
                  btn_run_icp.clicked.connect(self.execute_alignment)
                  align_layout.addWidget(self.lbl_align_info)
                  align_layout.addWidget(btn_start_align)
                  align_layout.addWidget(btn_run_icp)
                  align_group.setLayout(align_layout)
                  control_panel.addWidget(align_group)

                  margin_group = QGroupBox("3. Kole Çizimi & Derinlik (Ditching)")
                  margin_layout = QVBoxLayout()

                  jaw_select_layout = QHBoxLayout()
                  self.rb_upper = QRadioButton("Üst Çene")
                  self.rb_upper.setChecked(True)
                  self.rb_lower = QRadioButton("Alt Çene")
                  self.rb_upper.toggled.connect(lambda: self.set_active_jaw("upper"))
                  self.rb_lower.toggled.connect(lambda: self.set_active_jaw("lower"))
                  jaw_select_layout.addWidget(self.rb_upper)
                  jaw_select_layout.addWidget(self.rb_lower)
                  margin_layout.addLayout(jaw_select_layout)

                  margin_layout.addWidget(QLabel("Marjin Noktası Ekle: Sağ Tık"))

                  btn_smooth_margin = QPushButton("Marjin Hattını Pürüzsüzleştir (Spline)")
                  btn_smooth_margin.clicked.connect(self.update_spline_preview)
                  margin_layout.addWidget(btn_smooth_margin)

                  btn_clear_margin = QPushButton("Mevcut Çizimi Temizle")
                  btn_clear_margin.clicked.connect(self.clear_margin)
                  margin_layout.addWidget(btn_clear_margin)

                  depth_layout = QHBoxLayout()
                  depth_layout.addWidget(QLabel("Kole Derinliği (mm):"))
                  self.spin_depth = QDoubleSpinBox()
                  self.spin_depth.setRange(0.05, 2.00)
                  self.spin_depth.setSingleStep(0.05)
                  self.spin_depth.setValue(0.35)
                  depth_layout.addWidget(self.spin_depth)
                  margin_layout.addLayout(depth_layout)

                  width_layout = QHBoxLayout()
                  width_layout.addWidget(QLabel("Oluk Genişliği (mm):"))
                  self.spin_width = QDoubleSpinBox()
                  self.spin_width.setRange(0.10, 2.50)
                  self.spin_width.setSingleStep(0.10)
                  self.spin_width.setValue(0.60)
                  width_layout.addWidget(self.spin_width)
                  margin_layout.addLayout(width_layout)

                  btn_apply_ditch = QPushButton("Kole Derinliğini Uygula (Pürüzsüz Aç)")
                  btn_apply_ditch.setStyleSheet("background-color: #2e7d32; color: white; font-weight: bold; padding: 6px;")
                  btn_apply_ditch.clicked.connect(self.apply_ditching)
                  margin_layout.addWidget(btn_apply_ditch)

                  margin_group.setLayout(margin_layout)
                  control_panel.addWidget(margin_group)

                  export_group = QGroupBox("4. Exocad İçin Dışa Aktar")
                  export_layout = QVBoxLayout()
                  btn_export = QPushButton("Alt ve Üst Çeneyi Ayrı STL Kaydet")
                  btn_export.setStyleSheet("background-color: #1565c0; color: white; font-weight: bold; padding: 8px;")
                  btn_export.clicked.connect(self.export_for_exocad)
                  export_layout.addWidget(btn_export)
                  export_group.setLayout(export_layout)
                  control_panel.addWidget(export_group)

                  control_panel.addStretch()
                  layout.addLayout(control_panel, 1)

                  self.plotter = QtInteractor(self)
                  self.plotter.set_background("#1e1e1e")
                  self.plotter.add_axes()
                  layout.addWidget(self.plotter.interactor, 4)

                  self.plotter.enable_surface_point_picking(
                      callback=self.on_point_picked,
                      show_message=False,
                      use_picker=True,
                      left_clicking=False
                  )

              def set_active_jaw(self, jaw):
                  self.active_jaw = jaw
                  self.clear_margin()
                  self.render_meshes()

              def load_stl(self, jaw):
                  path, _ = QFileDialog.getOpenFileName(self, f"{jaw.upper()} STL Seç", "", "STL Files (*.stl)")
                  if not path:
                      return
                  mesh = pv.read(path)
                  mesh = mesh.compute_normals(auto_orient_normals=True)
                  if jaw == "upper":
                      self.upper_mesh = mesh
                  else:
                      self.lower_mesh = mesh
                  self.render_meshes()
                  self.plotter.reset_camera()

              def render_meshes(self):
                  self.plotter.clear()
                  self.plotter.add_axes()
                  if self.upper_mesh is not None:
                      opacity = 1.0 if self.active_jaw == "upper" or self.align_mode else 0.45
                      self.plotter.add_mesh(
                          self.upper_mesh, color="#f5e6d3", opacity=opacity,
                          smooth_shading=True, name="upper"
                      )
                  if self.lower_mesh is not None:
                      opacity = 1.0 if self.active_jaw == "lower" or self.align_mode else 0.45
                      self.plotter.add_mesh(
                          self.lower_mesh, color="#e0cbb4", opacity=opacity,
                          smooth_shading=True, name="lower"
                      )
                  self.update_spline_preview()

              def on_point_picked(self, point):
                  if point is None:
                      return
                  pt = np.array(point)

                  if self.align_mode:
                      if len(self.upper_align_pts) < 3:
                          self.upper_align_pts.append(pt)
                          actor = self.plotter.add_mesh(pv.Sphere(radius=0.4, center=pt), color="red")
                          self.align_actors.append(actor)
                          self.lbl_align_info.setText(f"Üst çenede {len(self.upper_align_pts)}/3 nokta seçildi. Alt çenede karşılıklarını seçin.")
                      elif len(self.lower_align_pts) < 3:
                          self.lower_align_pts.append(pt)
                          actor = self.plotter.add_mesh(pv.Sphere(radius=0.4, center=pt), color="blue")
                          self.align_actors.append(actor)
                          self.lbl_align_info.setText(f"Alt çenede {len(self.lower_align_pts)}/3 nokta seçildi. 'Kapanışı Kilitle'ye basın.")
                      return

                  self.margin_points.append(pt)
                  self.update_spline_preview()

              def update_spline_preview(self):
                  if self.margin_actor:
                      self.plotter.remove_actor(self.margin_actor)
                  if self.spline_actor:
                      self.plotter.remove_actor(self.spline_actor)

                  if len(self.margin_points) == 0:
                      return

                  pts = np.array(self.margin_points)
                  cloud = pv.PolyData(pts)
                  self.margin_actor = self.plotter.add_mesh(
                      cloud, color="#ff3d00", point_size=10, render_points_as_spheres=True
                  )

                  if len(pts) >= 4:
                      tck, u = splprep([pts[:, 0], pts[:, 1], pts[:, 2]], s=0.05, per=True)
                      u_new = np.linspace(0, 1, 300)
                      x_new, y_new, z_new = splev(u_new, tck)
                      spline_pts = np.column_stack((x_new, y_new, z_new))

                      target_mesh = self.upper_mesh if self.active_jaw == "upper" else self.lower_mesh
                      if target_mesh is not None:
                          tree = cKDTree(target_mesh.points)
                          _, idx = tree.query(spline_pts)
                          spline_pts = target_mesh.points[idx]

                      poly_line = pv.lines_from_points(spline_pts, close=True)
                      self.spline_actor = self.plotter.add_mesh(poly_line, color="#00e676", line_width=4)

              def clear_margin(self):
                  self.margin_points = []
                  if self.margin_actor:
                      self.plotter.remove_actor(self.margin_actor)
                  if self.spline_actor:
                      self.plotter.remove_actor(self.spline_actor)

              def apply_ditching(self):
                  target_mesh = self.upper_mesh if self.active_jaw == "upper" else self.lower_mesh
                  if target_mesh is None or len(self.margin_points) < 4:
                      QMessageBox.warning(self, "Uyarı", "Önce çene yükleyin ve en az 4 marjin noktası belirleyin.")
                      return

                  pts = np.array(self.margin_points)
                  tck, _ = splprep([pts[:, 0], pts[:, 1], pts[:, 2]], s=0.05, per=True)
                  u_new = np.linspace(0, 1, 500)
                  x_new, y_new, z_new = splev(u_new, tck)
                  curve_pts = np.column_stack((x_new, y_new, z_new))

                  depth = self.spin_depth.value()
                  width = self.spin_width.value()

                  center = np.mean(curve_pts, axis=0)
                  radial_dirs = curve_pts - center
                  radial_dirs /= np.linalg.norm(radial_dirs, axis=1, keepdims=True)
                  ditch_center_curve = curve_pts + radial_dirs * (width * 0.45)

                  tree = cKDTree(ditch_center_curve)
                  distances, _ = tree.query(target_mesh.points)

                  sigma = width / 2.2
                  displacement = depth * np.exp(-(distances * 2) / (2 * (sigma * 2)))

                  margin_tree = cKDTree(curve_pts)
                  _, m_idx = margin_tree.query(target_mesh.points)
                  vec_to_pt = target_mesh.points - curve_pts[m_idx]
                  is_outside = np.sum(vec_to_pt * radial_dirs[m_idx], axis=1) > 0
                  displacement[~is_outside] = 0.0

                  target_mesh = target_mesh.compute_normals(auto_orient_normals=True)
                  normals = target_mesh.point_normals
                  target_mesh.points -= normals * displacement[:, np.newaxis]
                  target_mesh = target_mesh.smooth(n_iter=15, relaxation_factor=0.05)

                  if self.active_jaw == "upper":
                      self.upper_mesh = target_mesh
                  else:
                      self.lower_mesh = target_mesh

                  self.clear_margin()
                  self.render_meshes()
                  QMessageBox.information(self, "Başarılı", f"{depth} mm derinliğinde pürüzsüz kole oluğu açıldı.")

              def start_alignment_mode(self):
                  if self.upper_mesh is None or self.lower_mesh is None:
                      QMessageBox.warning(self, "Uyarı", "Kapanış için hem alt hem üst çene yüklü olmalıdır.")
                      return
                  self.align_mode = True
                  self.upper_align_pts = []
                  self.lower_align_pts = []
                  for a in self.align_actors:
                      self.plotter.remove_actor(a)
                  self.align_actors = []
                  self.render_meshes()
                  self.lbl_align_info.setText("Üst çenede 3 referans noktaya sağ tıklayın (Kırmızı).")

              def execute_alignment(self):
                  if len(self.upper_align_pts) != 3 or len(self.lower_align_pts) != 3:
                      QMessageBox.warning(self, "Uyarı", "Üst çeneden 3, alt çeneden 3 nokta seçmelisiniz.")
                      return

                  src = np.array(self.lower_align_pts)
                  dst = np.array(self.upper_align_pts)

                  src_mean = src.mean(axis=0)
                  dst_mean = dst.mean(axis=0)
                  H = (src - src_mean).T @ (dst - dst_mean)
                  U, S, Vt = np.linalg.svd(H)
                  R = Vt.T @ U.T
                  if np.linalg.det(R) < 0:
                      Vt[2, :] *= -1
                      R = Vt.T @ U.T
                  t = dst_mean - R @ src_mean

                  T_4x4 = np.eye(4)
                  T_4x4[:3, :3] = R
                  T_4x4[:3, 3] = t

                  self.lower_mesh.transform(T_4x4, inplace=True)

                  self.align_mode = False
                  for a in self.align_actors:
                      self.plotter.remove_actor(a)
                  self.render_meshes()
                  self.lbl_align_info.setText("Durum: Seçilen 3 noktaya göre kapanış hizalandı.")
                  QMessageBox.information(self, "Kapanış Tamamlandı", "Alt ve üst çene seçilen referans noktalara göre hizalandı.")

              def export_for_exocad(self):
                  if self.upper_mesh is None and self.lower_mesh is None:
                      return
                  folder = QFileDialog.getExistingDirectory(self, "Exocad Çıktı Klasörünü Seçin")
                  if not folder:
                      return

                  if self.upper_mesh is not None:
                      self.upper_mesh.save(f"{folder}/Upper_Exocad_Ready.stl")
                  if self.lower_mesh is not None:
                      self.lower_mesh.save(f"{folder}/Lower_Exocad_Ready.stl")

                  QMessageBox.information(
                      self, "Dışa Aktarım Başarılı",
                      "Dosyalar Exocad koordinat sistemine uygun olarak kaydedildi:\n"
                      "- Upper_Exocad_Ready.stl\n- Lower_Exocad_Ready.stl"
                  )


          if _name_ == "_main_":
              app = QApplication(sys.argv)
              window = DentalPrepApp()
              window.show()
              sys.exit(app.exec_())
          '@ | Out-File -Encoding UTF8 app.py

      - name: Build EXE with PyInstaller
        run: pyinstaller --noconsole --onedir --name "ExocadPrepAssistant" --collect-all pyvista --collect-all pyvistaqt --collect-all scipy app.py

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: ExocadPrepAssistant-Win11
          path: dist/ExocadPrepAssistant
