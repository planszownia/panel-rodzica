# panel-rodzica
Panel dla rodziców do dodawania swoich dzieci i opłacania członkostwa w klubie małego stratega
<?php
/**
 * Planszownia Gracz 1.5.2
 * PROSTE POWIADOMIENIA WYPOŻYCZALNI
 *
 * Nie zmienia istniejących wypożyczeń.
 * Nie zmienia Strefy Rodzica.
 */

/* =========================================================
 * 1. TABELA POWIADOMIEŃ
 * ========================================================= */

function pw152_notifications_install() {

    global $wpdb;

    $table = $wpdb->prefix . 'pw_notifications';

    require_once ABSPATH . 'wp-admin/includes/upgrade.php';

    $charset = $wpdb->get_charset_collate();

    $sql = "CREATE TABLE {$table} (
        id bigint unsigned NOT NULL AUTO_INCREMENT,
        user_id bigint unsigned NOT NULL,
        loan_id bigint unsigned NOT NULL DEFAULT 0,
        type varchar(40) NOT NULL DEFAULT 'loan',
        message text NOT NULL,
        is_read tinyint(1) NOT NULL DEFAULT 0,
        created_at datetime NOT NULL,
        PRIMARY KEY (id),
        KEY user_id (user_id),
        KEY loan_id (loan_id),
        KEY is_read (is_read)
    ) {$charset};";

    dbDelta($sql);
}

add_action(
    'init',
    'pw152_notifications_install'
);


/* =========================================================
 * 2. DODAWANIE POWIADOMIENIA
 * ========================================================= */

function pw152_add_notification(
    $user_id,
    $message,
    $loan_id = 0,
    $type = 'loan'
) {

    $user_id = absint($user_id);
    $loan_id = absint($loan_id);
    $message = sanitize_text_field($message);
    $type    = sanitize_key($type);

    if (!$user_id || !$message) {
        return;
    }

    global $wpdb;

    $table =
        $wpdb->prefix . 'pw_notifications';

    $wpdb->insert(
        $table,
        array(
            'user_id'    => $user_id,
            'loan_id'    => $loan_id,
            'type'       => $type,
            'message'    => $message,
            'is_read'    => 0,
            'created_at' => current_time('mysql'),
        ),
        array(
            '%d',
            '%d',
            '%s',
            '%s',
            '%d',
            '%s',
        )
    );
}


/* =========================================================
 * 3. LICZBA NIEPRZECZYTANYCH
 * ========================================================= */

add_action(
    'wp_ajax_pw152_notifications_count',
    function () {

        if (!is_user_logged_in()) {
            wp_send_json_error();
        }

        global $wpdb;

        $count = $wpdb->get_var(
            $wpdb->prepare(
                "SELECT COUNT(*)
                 FROM {$wpdb->prefix}pw_notifications
                 WHERE user_id = %d
                 AND is_read = 0",
                get_current_user_id()
            )
        );

        wp_send_json_success(
            array(
                'count' => (int) $count
            )
        );
    }
);


/* =========================================================
 * 4. LISTA POWIADOMIEŃ
 * ========================================================= */

add_action(
    'wp_ajax_pw152_notifications_list',
    function () {

        if (!is_user_logged_in()) {
            wp_send_json_error();
        }

        global $wpdb;

        $rows = $wpdb->get_results(
            $wpdb->prepare(
                "SELECT *
                 FROM {$wpdb->prefix}pw_notifications
                 WHERE user_id = %d
                 ORDER BY created_at DESC, id DESC
                 LIMIT 30",
                get_current_user_id()
            )
        );

        ob_start();

        ?>

        <div class="pw152-notifications-list">

            <?php if (empty($rows)) : ?>

                <div style="padding:10px;opacity:.6;">
                    Brak powiadomień.
                </div>

            <?php else : ?>

                <?php foreach ($rows as $row) : ?>

                    <div
                        class="pw152-notification <?php
                            echo $row->is_read
                                ? 'is-read'
                                : 'is-new';
                        ?>"
                        data-id="<?php
                            echo (int) $row->id;
                        ?>"
                        style="
                            padding:10px;
                            margin-bottom:7px;
                            border-radius:8px;
                            background:<?php
                                echo $row->is_read
                                    ? '#f7f7f7'
                                    : '#fff3cd';
                            ?>;
                        "
                    >

                        <div>
                            <?php
                            echo esc_html(
                                $row->message
                            );
                            ?>
                        </div>

                        <small style="opacity:.6;">
                            <?php
                            echo esc_html(
                                mysql2date(
                                    'd.m.Y H:i',
                                    $row->created_at
                                )
                            );
                            ?>
                        </small>

                    </div>

                <?php endforeach; ?>

            <?php endif; ?>

        </div>

        <?php

        wp_send_json_success(
            array(
                'html' =>
                    ob_get_clean()
            )
        );
    }
);


/* =========================================================
 * 5. OZNACZ JAKO PRZECZYTANE
 * ========================================================= */

add_action(
    'wp_ajax_pw152_notifications_read',
    function () {

        if (!is_user_logged_in()) {
            wp_send_json_error();
        }

        global $wpdb;

        $id = absint(
            $_POST['id'] ?? 0
        );

        if ($id) {

            $wpdb->update(
                $wpdb->prefix . 'pw_notifications',
                array(
                    'is_read' => 1
                ),
                array(
                    'id'      => $id,
                    'user_id' =>
                        get_current_user_id()
                ),
                array(
                    '%d'
                ),
                array(
                    '%d',
                    '%d'
                )
            );

        } else {

            /*
             * Brak ID = oznacz wszystkie jako przeczytane.
             */

            $wpdb->update(
                $wpdb->prefix . 'pw_notifications',
                array(
                    'is_read' => 1
                ),
                array(
                    'user_id' =>
                        get_current_user_id(),
                    'is_read' => 0
                ),
                array(
                    '%d'
                ),
                array(
                    '%d',
                    '%d'
                )
            );
        }

        wp_send_json_success();
    }
);


/* =========================================================
 * 6. WIDOCZNY LICZNIK
 * ========================================================= */

add_action(
    'wp_footer',
    function () {

        if (!is_user_logged_in()) {
            return;
        }

        ?>

        <style>

        .pw152-notification-badge {
            display:inline-flex;
            align-items:center;
            justify-content:center;

            min-width:20px;
            height:20px;

            padding:0 6px;

            margin-left:6px;

            border-radius:20px;

            background:#e53935;
            color:#fff;

            font-size:11px;
            font-weight:700;

            line-height:20px;
        }

        .pw152-notification-bell {
            cursor:pointer;
            position:relative;
        }

        .pw152-notification-box {
            display:none;

            position:absolute;
            z-index:99999;

            width:320px;
            max-width:90vw;

            padding:12px;

            background:#fff;

            border:1px solid #ddd;
            border-radius:10px;

            box-shadow:
                0 8px 25px rgba(0,0,0,.15);
        }

        .pw152-notification-box.is-open {
            display:block;
        }

        </style>


        <script>

        (function () {

            function pw152NotificationAjax(
                data
            ) {

                data.nonce =
                    window.PWGracz
                        ? PWGracz.nonce
                        : '';

                return fetch(
                    window.PWGracz &&
                    PWGracz.ajax
                        ? PWGracz.ajax
                        : '<?php
                            echo esc_url(
                                admin_url(
                                    'admin-ajax.php'
                                )
                            );
                        ?>',
                    {
                        method:'POST',

                        headers:{
                            'Content-Type':
                                'application/x-www-form-urlencoded; charset=UTF-8'
                        },

                        body:
                            new URLSearchParams(
                                data
                            ).toString()
                    }
                )
                .then(function(response){

                    return response.text();

                })
                .then(function(text){

                    try {
                        return JSON.parse(text);
                    } catch(e) {

                        console.error(
                            'Powiadomienia:',
                            text
                        );

                        return {
                            success:false
                        };
                    }

                });
            }


            function pw152GetCount() {

                pw152NotificationAjax({
                    action:
                        'pw152_notifications_count'
                })
                .then(function(result){

                    if (
                        !result ||
                        !result.success
                    ) {
                        return;
                    }

                    var count =
                        parseInt(
                            result.data.count,
                            10
                        ) || 0;

                    document
                        .querySelectorAll(
                            '.pw152-notification-badge'
                        )
                        .forEach(function(badge){

                            if (count > 0) {

                                badge.textContent =
                                    count;

                                badge.style.display =
                                    'inline-flex';

                            } else {

                                badge.style.display =
                                    'none';
                            }

                        });

                });
            }


            function pw152CreateBell() {

                if (
                    document.querySelector(
                        '.pw152-notification-bell'
                    )
                ) {
                    return;
                }

                /*
                 * Szukamy zakładki/panelu Gracza.
                 */
                var target =
                    document.querySelector(
                        '[data-pw-panel]'
                    );

                if (!target) {
                    return;
                }

                var bell =
                    document.createElement(
                        'div'
                    );

                bell.className =
                    'pw152-notification-bell';

                bell.style.cssText =
                    'margin:10px 0;position:relative;';

                bell.innerHTML =
                    '<button type="button" class="pw-btn">' +
                        '🔔 Powiadomienia' +
                        '<span class="pw152-notification-badge" style="display:none;"></span>' +
                    '</button>' +

                    '<div class="pw152-notification-box">' +

                        '<div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:10px;">' +

                            '<strong>🔔 Powiadomienia</strong>' +

                            '<button type="button" class="pw152-notifications-read-all" style="border:0;background:none;cursor:pointer;">' +
                                'Oznacz jako przeczytane' +
                            '</button>' +

                        '</div>' +

                        '<div class="pw152-notifications-content">' +
                            'Ładowanie…' +
                        '</div>' +

                    '</div>';

                target.prepend(bell);


                var button =
                    bell.querySelector(
                        'button.pw-btn'
                    );

                var box =
                    bell.querySelector(
                        '.pw152-notification-box'
                    );


                button.addEventListener(
                    'click',
                    function(e){

                        e.preventDefault();

                        box.classList.toggle(
                            'is-open'
                        );

                        if (
                            box.classList.contains(
                                'is-open'
                            )
                        ) {

                            pw152LoadNotifications();

                        }

                    }
                );


                bell.querySelector(
                    '.pw152-notifications-read-all'
                ).addEventListener(
                    'click',
                    function(){

                        pw152NotificationAjax({
                            action:
                                'pw152_notifications_read'
                        })
                        .then(function(){

                            pw152LoadNotifications();

                            pw152GetCount();

                        });

                    }
                );

            }


            function pw152LoadNotifications() {

                var content =
                    document.querySelector(
                        '.pw152-notifications-content'
                    );

                if (!content) {
                    return;
                }

                pw152NotificationAjax({
                    action:
                        'pw152_notifications_list'
                })
                .then(function(result){

                    if (
                        result &&
                        result.success &&
                        result.data &&
                        result.data.html
                    ) {

                        content.innerHTML =
                            result.data.html;
                    }

                });

            }


            document.addEventListener(
                'DOMContentLoaded',
                function(){

                    pw152CreateBell();

                    pw152GetCount();

                }
            );


            /*
             * Panel Gracza może być przeładowywany AJAX-em.
             */

            setInterval(
                function(){

                    pw152CreateBell();

                    pw152GetCount();

                },
                5000
            );

        })();

        </script>

        <?php
    }
);
