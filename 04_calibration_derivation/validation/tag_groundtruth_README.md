# Tag ground truth (TagSLAM) vs. OpenVINS: recording VID_20240618_170514_010

The question: is the VIO scale right? To answer it, I built an independent, metrically scaled ground truth from
60 floor tags and compared OpenVINS against it.
* Tags: tag36h11, IDs 0–59, black square 0.160 m, lying at random positions and orientations on the floor.
* Recording: handheld walk over the tags, 155 s.

## Result

| OpenVINS variant | Sim(3) scale correction | OpenVINS too large by | ATE RMSE SE(3) | ATE RMSE Sim(3) |
|---|---|---|---|---|
| **ov_calibnoise**: IMU noise = calibration (Allan) × 5 | **0.959** | **4.3 %** | **0.23 m** | 0.23 m |
| ov_flightnoise: IMU noise = in-flight values (prop vibration) | 0.592 | 69 % | 1.12 m | 0.80 m |

The ground-truth path (from OpenVINS initialization at t = 15.6 s) is 41.2 m. Both runs use the same Basalt
calibration, with extrinsics and time offset fixed and dynamic init.
* Top view: `eval/topview.png`. evo results: `eval/*.zip`.
* `-a` is SE(3) alignment; `-as` is Sim(3).

* **The VIO chain itself is fine.** With an IMU noise model that matches the recording, OpenVINS is within 4 % of the
  tag ground truth in scale, and 0.23 m ATE over 41 m.
* **A noise model much too large for the data ruins the scale.** The in-flight values (25× static, needed for the
  prop vibration in flight) let the IMU constrain almost nothing on a vibration-free handheld walk, so the scale
  drifts (69 %).
* **The "~1.37" from the plan is not a VIO result.** It is 1/0.727, the accelerometer scale of the first, rejected
  `basalt_calibrate_vi` run (`opt_imu_scale`, gravity free). The final calibration has the accelerometer scale fixed
  to 1 and |g| = 9.798 m/s². In this recording, |a| at the start is 9.746 m/s².

## Ground-truth quality (TagSLAM)

* **All 60 tags are in the map.** The map is 4.7 × 4.5 m. One false detection (ID 564) is ignored, because it isn't in the config.
* **Reprojection error** over 154,936 corners at 2880 px: median 0.59 px, mean 0.75, p90 1.39, p99 3.3 px.
  Tag 12 is the worst at 2.6 px median; all others are ≤ 0.9 px.
* **Flatness**, an independent check, since TagSLAM was not told the tags lie on a floor:
  * all tag centres are within 2.8 mm (std) of one plane, max 8 mm
  * the tag normals are within 0.36° (median) and 0.83° (max) of each other.

## Scale check with the laser rangefinder (still to do)

Measure these centre-to-centre distances and compare them with the map. A few mm difference means the ground truth is right.

| tags | map distance |
|---|---|
| 0 ↔ 51 | 5961 mm |
| 23 ↔ 44 | 5464 mm |
| 10 ↔ 47 | 4325 mm |

This only works if the tags still lie exactly as they did on 18 June 2024.

## How it was made

Everything except TagSLAM itself runs from `insv_imu_viz/` with its `.venv`.

1. Bag: ROS2/Jazzy, 2880 px, every 3rd stereo pair (10 fps), each lens masked beyond 80° from its axis.
   ```
   .venv/bin/python vio/export_rosbag.py --rec VID_20240618_170514_010 --ros2 --every 3 --mask-theta 80 \
       --calib ../betafpv_smo_360_calibration/betafpv_smo_360_calibration.json \
       --out /run/media/niklas/EDDF-B044/tagslam/rec010_bag_th80
   ```
   The mask is needed because TagSLAM's equidistant model uses θ = atan(r) on the image plane, so it cannot represent
   rays at or beyond 90°. Without the mask, about 2 % of the corners were at 90–101°, and GTSAM aborted with
   "IndeterminantLinearSystem".
2. Configs: `cameras.yaml`, `camera_poses.yaml`, `tagslam.yaml`.
   ```
   .venv/bin/python vio/make_tagslam_config.py --calib ../betafpv_smo_360_calibration/betafpv_smo_360_calibration.json \
       --out ../tagslam_gt/rec010 --tag-size 0.160 --anchor 15 --tag-ids 0-59
   ```
   * Intrinsics: kb4 = TagSLAM `equidistant`, copied 1:1.
   * cam1 pose in the rig (= cam0) frame: T_c0_c1 = T_i_c0⁻¹·T_i_c1.
   * Anchor: tag 15, because it is seen in the first frame. TagSLAM drops frames until a tag with a known pose is visible.
   * `optimizer_mode` must be `fast`: `full` crashes in GTSAM in this ROS2 branch.
3. TagSLAM: ros2 branch, built in `~/tagslam_ws` in the `ros2-24-04` distrobox.
   * Packages it needs: ros-jazzy-gtsam, -apriltag-detector(-umich/-mit), -apriltag-msgs, -flex-sync.
   * Build with `PATH=/usr/bin:/bin`, so it doesn't pick up the host's ~/.local python.
   ```
   ulimit -c 0     # a crash otherwise writes a multi-GB core dump to the full system disk
   ros2 run tagslam tagslam_from_bag --ros-args -p cameras:=cameras.yaml -p tagslam_config:=tagslam.yaml \
       -p camera_poses:=camera_poses.yaml -p in_bag:=<bag> -p out_bag:=out/out_bag -p output_directory:=out \
       -p max_number_of_frames:=100000
   ```
   Quirks of this branch:
   * `max_number_of_frames` must be set; the default 0 stops immediately.
   * The replay that writes `out/out.bag` takes ~20 min with the default `publish_covariance: true`.
   * Two runs at once need different `ROS_DOMAIN_ID`s.
4. Ground-truth trajectory of the IMU (T_map_imu = T_map_cam0 · T_i_c0⁻¹) and tag map:
   `vio/tagslam_to_tum.py` → `out/tagslam_imu_tum.txt`, `out/tag_map.csv`.
5. OpenVINS: data on the external drive, `/run/media/niklas/EDDF-B044/vio_work/VID_20240618_170514_010/`.
   * Static masks come from `calib_from_data.py --t0 2 --t1 60`, plus a mask of the operator (dark region of the
     median image).
   * Config: `make_openvins_config.py --basalt ... --imu-noise calib --imu-noise-scale 5`
     `--set init_dyn_use=true try_zupt=false ...`. The camera is never still, so dynamic init is needed.
6. evo: `evo_ape tum out/tagslam_imu_tum.txt <ov_traj> -a|-as --t_offset 0.009516 --t_max_diff 0.01`.
   OpenVINS stamps are in the IMU clock (camera time − 9.5 ms).
